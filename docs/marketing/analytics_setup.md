"""
Credit Management System for Billing Operations
================================================

Production-grade implementation for managing credit grants, code redemptions,
and invoice credit applications within the Amberflo billing pipeline.

This module provides the core business logic for handling promotional credits,
trial balances, make-good adjustments, and bounty rewards with full audit trail,
error handling, and observability.

Version: 1.0.0
Author: Datum Engineering
"""

import asyncio
import hashlib
import hmac
import json
import logging
import re
import secrets
import time
import uuid
from dataclasses import dataclass, field, asdict
from datetime import datetime, timedelta, timezone
from decimal import Decimal, ROUND_HALF_UP
from enum import Enum, auto
from functools import lru_cache, wraps
from typing import (
    Any,
    Callable,
    Dict,
    List,
    Optional,
    Set,
    Tuple,
    TypeVar,
    Union,
    cast,
)
from urllib.parse import urlparse

import aiohttp
import stripe
from pydantic import (
    BaseModel,
    Field,
    ValidationError,
    validator,
    root_validator,
    ConfigDict,
)
from prometheus_client import Counter, Histogram, Gauge
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
    before_sleep_log,
)

# ---------------------------------------------------------------------------
# Constants and Configuration
# ---------------------------------------------------------------------------

logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# Prometheus metrics
CREDIT_GRANT_COUNTER = Counter(
    "credit_grants_total",
    "Total number of credit grants created",
    ["grant_type", "source"],
)
CODE_REDEMPTION_COUNTER = Counter(
    "code_redemptions_total",
    "Total number of code redemptions",
    ["redemption_type"],
)
INVOICE_CREDIT_APPLIED_COUNTER = Counter(
    "invoice_credit_applied_total",
    "Total number of invoices with credit applied",
    ["credit_fully_covered"],
)
CREDIT_AMOUNT_GAUGE = Gauge(
    "active_credit_balance_cents",
    "Current active credit balance in cents",
    ["account_id"],
)
CREDIT_OPERATION_DURATION = Histogram(
    "credit_operation_duration_seconds",
    "Duration of credit operations",
    ["operation_type"],
)

# Security constants
MINIMUM_CODE_LENGTH = 12
MAXIMUM_CODE_LENGTH = 64
CODE_CHARACTER_SET = "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
MAXIMUM_NOTE_LENGTH = 1000
MAXIMUM_AMOUNT_CENTS = 10_000_000  # $100,000
MINIMUM_AMOUNT_CENTS = 1  # $0.01
DEFAULT_EXPIRY_DAYS = 365
RATE_LIMIT_WINDOW_SECONDS = 60
RATE_LIMIT_MAX_REQUESTS = 100

# ---------------------------------------------------------------------------
# Enums and Types
# ---------------------------------------------------------------------------

class GrantType(str, Enum):
    """Types of credit grants supported by the system."""
    TRIAL = "trial"
    PROMOTIONAL = "promotional"
    MAKE_GOOD = "make_good"
    BOUNTY = "bounty"


class GrantSource(str, Enum):
    """Sources from which credit grants can originate."""
    STAFF_PORTAL = "staff_portal"
    CODE_REDEMPTION = "code_redemption"
    AUTOMATED = "automated"


class RedemptionType(str, Enum):
    """Types of code redemption."""
    ONE_TIME = "one_time"
    MULTI_USE = "multi_use"


class CreditCodeStatus(str, Enum):
    """Status of a credit redemption code."""
    ACTIVE = "active"
    EXPIRED = "expired"
    REVOKED = "revoked"
    EXHAUSTED = "exhausted"


class InvoiceStatus(str, Enum):
    """Status of an invoice in the billing system."""
    DRAFT = "draft"
    OPEN = "open"
    PAID = "paid"
    VOID = "void"
    UNCOLLECTIBLE = "uncollectible"


# ---------------------------------------------------------------------------
# Custom Exceptions
# ---------------------------------------------------------------------------

class CreditManagementError(Exception):
    """Base exception for credit management system."""
    pass


class AccountNotFoundError(CreditManagementError):
    """Raised when a billing account is not found."""
    pass


class InsufficientCreditError(CreditManagementError):
    """Raised when there is insufficient credit for an operation."""
    pass


class InvalidCodeError(CreditManagementError):
    """Raised when a redemption code is invalid."""
    pass


class CodeAlreadyRedeemedError(CreditManagementError):
    """Raised when a one-time code has already been redeemed."""
    pass


class CodeExpiredError(CreditManagementError):
    """Raised when a redemption code has expired."""
    pass


class CodeExhaustedError(CreditManagementError):
    """Raised when a multi-use code has reached its maximum redemptions."""
    pass


class CreditExpiredError(CreditManagementError):
    """Raised when a credit grant has expired."""
    pass


class RateLimitExceededError(CreditManagementError):
    """Raised when rate limit is exceeded."""
    pass


class ValidationError(CreditManagementError):
    """Raised when input validation fails."""
    pass


# ---------------------------------------------------------------------------
# Pydantic Models for Data Validation
# ---------------------------------------------------------------------------

class CreditGrantCreate(BaseModel):
    """Schema for creating a new credit grant."""
    model_config = ConfigDict(frozen=True, extra="forbid")

    account_id: str = Field(..., description="UUID of the billing account")
    amount_cents: int = Field(
        ...,
        ge=MINIMUM_AMOUNT_CENTS,
        le=MAXIMUM_AMOUNT_CENTS,
        description="Amount in cents (e.g., 5000 = $50.00)",
    )
    grant_type: GrantType = Field(..., description="Type of credit grant")
    source: GrantSource = Field(..., description="Source of the grant")
    note: Optional[str] = Field(
        None,
        max_length=MAXIMUM_NOTE_LENGTH,
        description="Free-text note for audit trail",
    )
    expiry_date: Optional[datetime] = Field(
        None,
        description="Optional expiry date for the credit",
    )
    created_by: str = Field(
        ...,
        min_length=3,
        max_length=255,
        description="Email or identifier of the creator",
    )

    @validator("account_id")
    def validate_account_id(cls, v: str) -> str:
        """Validate account ID format."""
        try:
            uuid.UUID(v)
        except ValueError as exc:
            raise ValueError(f"Invalid account_id format: {v}") from exc
        return v

    @validator("expiry_date")
    def validate_expiry_date(cls, v: Optional[datetime]) -> Optional[datetime]:
        """Ensure expiry date is in the future if provided."""
        if v and v <= datetime.now(timezone.utc):
            raise ValueError("Expiry date must be in the future")
        return v

    @validator("note")
    def sanitize_note(cls, v: Optional[str]) -> Optional[str]:
        """Sanitize note to prevent injection attacks."""
        if v:
            # Remove any HTML/script tags
            v = re.sub(r'<[^>]*>', '', v)
            # Trim whitespace
            v = v.strip()
        return v


class CreditCodeCreate(BaseModel):
    """Schema for creating a new credit redemption code."""
    model_config = ConfigDict(frozen=True, extra="forbid")

    amount_cents: int = Field(
        ...,
        ge=MINIMUM_AMOUNT_CENTS,
        le=MAXIMUM_AMOUNT_CENTS,
        description="Amount in cents the code is worth",
    )
    redemption_type: RedemptionType = Field(
        ...,
        description="Whether the code is one-time or multi-use",
    )
    max_redemptions: Optional[int] = Field(
        None,
        ge=1,
        le=1000,
        description="Maximum number of redemptions for multi-use codes",
    )
    campaign: Optional[str] = Field(
        None,
        max_length=100,
        description="Campaign identifier for tracking",
    )
    expiry_date: Optional[datetime] = Field(
        None,
        description="Optional expiry date for the code",
    )
    created_by: str = Field(
        ...,
        min_length=3,
        max_length=255,
        description="Email or identifier of the creator",
    )

    @validator("max_redemptions")
    def validate_max_redemptions(cls, v: Optional[int], values: Dict[str, Any]) -> Optional[int]:
        """Validate max_redemptions based on redemption type."""
        if values.get("redemption_type") == RedemptionType.MULTI_USE and v is None:
            raise ValueError("max_redemptions is required for multi-use codes")
        if values.get("redemption_type") == RedemptionType.ONE_TIME and v is not None:
            raise ValueError("max_redemptions should not be set for one-time codes")
        return v


class CodeRedemption(BaseModel):
    """Schema for redeeming a credit code."""
    model_config = ConfigDict(frozen=True, extra="forbid")

    code: str = Field(
        ...,
        min_length=MINIMUM_CODE_LENGTH,
        max_length=MAXIMUM_CODE_LENGTH,
        description="The redemption code",
    )
    account_id: str = Field(..., description="UUID of the billing account")
    redeemed_by: str = Field(
        ...,
        min_length=3,
        max_length=255,
        description="Email or identifier of the redeemer",
    )

    @validator("code")
    def validate_code_format(cls, v: str) -> str:
        """Validate code format and sanitize."""
        # Remove any whitespace and convert to uppercase
        v = v.strip().upper()
        # Validate characters
        if not all(c in CODE_CHARACTER_SET for c in v):
            raise ValueError("Code contains invalid characters")
        return v

    @validator("account_id")
    def validate_account_id(cls, v: str) -> str:
        """Validate account ID format."""
        try:
            uuid.UUID(v)
        except ValueError as exc:
            raise ValueError(f"Invalid account_id format: {v}") from exc
        return v


# ---------------------------------------------------------------------------
# Rate Limiter
# ---------------------------------------------------------------------------

class RateLimiter:
    """Token bucket rate limiter for API endpoints."""

    def __init__(self, max_requests: int = RATE_LIMIT_MAX_REQUESTS, window_seconds: int = RATE_LIMIT_WINDOW_SECONDS):
        """Initialize rate limiter with token bucket algorithm.

        Args:
            max_requests: Maximum number of requests allowed in the window
            window_seconds: Time window in seconds

        Raises:
            ValueError: If parameters are invalid
        """
        if max_requests <= 0:
            raise ValueError("max_requests must be positive")
        if window_seconds <= 0:
            raise ValueError("window_seconds must be positive")

        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self._buckets: Dict[str, Tuple[int, float]] = {}  # key -> (tokens, last_refill_time)
        self._lock = asyncio.Lock()

    async def check_rate_limit(self, key: str) -> bool:
        """Check if request is within rate limit.

        Args:
            key: Unique identifier for the rate limit bucket

        Returns:
            True if request is allowed, False otherwise

        Raises:
            RateLimitExceededError: If rate limit is exceeded
        """
        async with self._lock:
            now = time.time()
            tokens, last_refill = self._buckets.get(key, (self.max_requests, now))

            # Calculate tokens to add based on time elapsed
            elapsed = now - last_refill
            tokens_to_add = int(elapsed * (self.max_requests / self.window_seconds))
            tokens = min(self.max_requests, tokens + tokens_to_add)

            if tokens <= 0:
                raise RateLimitExceededError(f"Rate limit exceeded for key: {key}")

            tokens -= 1
            self._buckets[key] = (tokens, now)
            return True


# ---------------------------------------------------------------------------
# Credit Management Service
# ---------------------------------------------------------------------------

class CreditManagementService:
    """Core service for managing credit grants, codes, and invoice applications.

    This service handles all credit-related operations including:
    - Creating and managing credit grants
    - Generating and redeeming credit codes
    - Applying credits to invoices
    - Maintaining audit trails
    """

    def __init__(
        self,
        amberflo_client: Any,
        stripe_client: Any,
        rate_limiter: Optional[RateLimiter] = None,
    ):
        """Initialize the credit management service.

        Args:
            amberflo_client: Client for Amberflo metering API
            stripe_client: Client for Stripe payment processing
            rate_limiter: Optional rate limiter instance

        Raises:
            ValueError: If required clients are None
        """
        if amberflo_client is None:
            raise ValueError("amberflo_client is required")
        if stripe_client is None:
            raise ValueError("stripe_client is required")

        self._amberflo = amberflo_client
        self._stripe = stripe_client
        self._rate_limiter = rate_limiter or RateLimiter()
        self._logger = logging.getLogger(f"{__name__}.{self.__class__.__name__}")

    async def create_credit_grant(self, grant_data: CreditGrantCreate) -> Dict[str, Any]:
        """Create a new credit grant for a billing account.

        Args:
            grant_data: Validated credit grant creation data

        Returns:
            Dictionary containing the created grant details

        Raises:
            AccountNotFoundError: If the billing account doesn't exist
            ValidationError: If input validation fails
            CreditManagementError: For other credit management errors
        """
        operation_start = time.time()
        try:
            # Validate input
            if not isinstance(grant_data, CreditGrantCreate):
                raise ValidationError("Invalid grant data type")

            # Check rate limit
            await self._rate_limiter.check_rate_limit(f"create_grant:{grant_data.created_by}")

            # Verify account exists
            account = await self._get_billing_account(grant_data.account_id)
            if account is None:
                raise AccountNotFoundError(f"Billing account not found: {grant_data.account_id}")

            # Create the credit grant in Amberflo
            grant_payload = {
                "account_id": grant_data.account_id,
                "amount_cents": grant_data.amount_cents,
                "grant_type": grant_data.grant_type.value,
                "source": grant_data.source.value,
                "note": grant_data.note,
                "expiry_date": grant_data.expiry_date.isoformat() if grant_data.expiry_date else None,
                "created_by": grant_data.created_by,
                "created_at": datetime.now(timezone.utc).isoformat(),
                "grant_id": str(uuid.uuid4()),
            }

            result = await self._amberflo.create_prepaid_credit_grant(grant_payload)

            # Update metrics
            CREDIT_GRANT_COUNTER.labels(
                grant_type=grant_data.grant_type.value,
                source=grant_data.source.value,
            ).inc()

            CREDIT_AMOUNT_GAUGE.labels(
                account_id=grant_data.account_id
            ).set(await self._get_active_balance(grant_data.account_id))

            self._logger.info(
                "Credit grant created",
                extra={
                    "grant_id": result.get("grant_id"),
                    "account_id": grant_data.account_id,
                    "amount_cents": grant_data.amount_cents,
                    "grant_type": grant_data.grant_type.value,
                    "created_by": grant_data.created_by,
                },
            )

            return result

        except AccountNotFoundError:
            raise
        except ValidationError:
            raise
        except RateLimitExceededError:
            raise
        except Exception as exc:
            self._logger.error(
                "Failed to create credit grant",
                extra={
                    "account_id": grant_data.account_id,
                    "amount_cents": grant_data.amount_cents,
                    "error": str(exc),
                },
                exc_info=True,
            )
            raise CreditManagementError(f"Failed to create credit grant: {exc}") from exc
        finally:
            CREDIT_OPERATION_DURATION.labels(
                operation_type="create_credit_grant"
            ).observe(time.time() - operation_start)

    async def create_credit_code(self, code_data: CreditCodeCreate) -> Dict[str, Any]:
        """Create a new credit redemption code.

        Args:
            code_data: Validated credit code creation data

        Returns:
            Dictionary containing the created code details

        Raises:
            ValidationError: If input validation fails
            CreditManagementError: For other credit management errors
        """
        operation_start = time.time()
        try:
            # Validate input
            if not isinstance(code_data, CreditCodeCreate):
                raise ValidationError("Invalid code data type")

            # Check rate limit
            await self._rate_limiter.check_rate_limit(f"create_code:{code_data.created_by}")

            # Generate unique code
            code = await self._generate_unique_code()

            # Create the code record
            code_record = {
                "code": code,
                "amount_cents": code_data.amount_cents,
                "redemption_type": code_data.redemption_type.value,
                "max_redemptions": code_data.max_redemptions,
                "current_redemptions": 0,
                "campaign": code_data.campaign,
                "expiry_date": code_data.expiry_date.isoformat() if code_data.expiry_date else None,
                "created_by": code_data.created_by,
                "created_at": datetime.now(timezone.utc).isoformat(),
                "status": CreditCodeStatus.ACTIVE.value,
                "code_id": str(uuid.uuid4()),
            }

            # Store in database (implementation depends on storage backend)
            result = await self._store_credit_code(code_record)

            self._logger.info(
                "Credit code created",
                extra={
                    "code_id": result.get("code_id"),
                    "amount_cents": code_data.amount_cents,
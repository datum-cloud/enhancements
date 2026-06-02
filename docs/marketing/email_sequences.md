"""
Email Sequences: Credit Ledger Launch – Amberflo Integration

This module defines and manages email sequences for the Credit Ledger Launch,
including internal staff onboarding and end-user notifications. It provides
comprehensive error handling, type annotations, logging, input validation,
and performance optimization.
"""

import logging
import re
from datetime import datetime, timedelta
from enum import Enum
from typing import Dict, List, Optional, Any, Union, Set, Tuple
from dataclasses import dataclass, field
from abc import ABC, abstractmethod
import json
import hashlib
from functools import lru_cache

# Configure logging
logger = logging.getLogger(__name__)
logger.setLevel(logging.INFO)

# Add handler if not already configured
if not logger.handlers:
    handler = logging.StreamHandler()
    handler.setFormatter(logging.Formatter(
        '%(asctime)s - %(name)s - %(levelname)s - %(message)s'
    ))
    logger.addHandler(handler)


class EmailTrigger(Enum):
    """Enumeration of possible email triggers."""
    STAFF_GRANTED_PERMISSIONS = "staff_granted_permissions"
    CREDIT_GRANT_APPLIED = "credit_grant_applied"
    CODE_REDEEMED = "code_redeemed"
    CREDIT_EXPIRING_SOON = "credit_expiring_soon"
    CREDIT_EXPIRED = "credit_expired"


class EmailSequence(Enum):
    """Enumeration of email sequences."""
    INTERNAL_STAFF_ONBOARDING = "internal_staff_onboarding"
    END_USER_NOTIFICATION = "end_user_notification"


class EmailTemplateError(Exception):
    """Custom exception for email template errors."""
    pass


class ValidationError(Exception):
    """Custom exception for validation errors."""
    pass


class SecurityError(Exception):
    """Custom exception for security-related errors."""
    pass


@dataclass
class EmailTemplate:
    """
    Represents an email template with subject and body.
    
    Attributes:
        subject: Email subject line
        body: Email body content
        template_variables: Set of required template variables
        version: Template version for tracking changes
        created_at: Timestamp of template creation
    """
    subject: str
    body: str
    template_variables: Set[str] = field(default_factory=set)
    version: str = field(default_factory=lambda: datetime.now().isoformat())
    created_at: datetime = field(default_factory=datetime.now)
    
    def __post_init__(self) -> None:
        """Extract template variables from subject and body."""
        try:
            self.template_variables = self._extract_variables()
            self._validate_template_content()
        except Exception as e:
            logger.error(f"Failed to initialize EmailTemplate: {e}")
            raise EmailTemplateError(f"Template initialization failed: {e}") from e
    
    def _validate_template_content(self) -> None:
        """
        Validate template content for security and correctness.
        
        Raises:
            ValidationError: If template content is invalid
            SecurityError: If template contains malicious content
        """
        if not self.subject or not self.body:
            raise ValidationError("Subject and body cannot be empty")
        
        if len(self.subject) > 500:
            raise ValidationError("Subject exceeds maximum length of 500 characters")
        
        if len(self.body) > 100000:
            raise ValidationError("Body exceeds maximum length of 100000 characters")
        
        # Security check: prevent template injection
        malicious_patterns = [
            r'<script[^>]*>.*?</script>',  # XSS attempts
            r'javascript:',  # JavaScript protocol
            r'on\w+\s*=',  # Event handlers
            r'{{.*}}.*{{.*}}',  # Nested templates
        ]
        
        for pattern in malicious_patterns:
            if re.search(pattern, self.subject, re.IGNORECASE) or \
               re.search(pattern, self.body, re.IGNORECASE):
                raise SecurityError(f"Template contains potentially malicious content: {pattern}")
    
    def _extract_variables(self) -> Set[str]:
        """
        Extract template variables from subject and body.
        
        Returns:
            Set of variable names found in the template
        """
        try:
            pattern = r'\{\{(\w+)\}\}'
            subject_vars = set(re.findall(pattern, self.subject))
            body_vars = set(re.findall(pattern, self.body))
            return subject_vars | body_vars
        except Exception as e:
            logger.error(f"Failed to extract template variables: {e}")
            raise EmailTemplateError(f"Variable extraction failed: {e}") from e
    
    def validate_variables(self, variables: Dict[str, str]) -> bool:
        """
        Validate that all required variables are provided and valid.
        
        Args:
            variables: Dictionary of variable names to values
            
        Returns:
            True if all required variables are present and valid
            
        Raises:
            ValidationError: If required variables are missing or invalid
            SecurityError: If variables contain malicious content
        """
        if not isinstance(variables, dict):
            raise ValidationError("Variables must be a dictionary")
        
        # Check for required variables
        missing_vars = self.template_variables - set(variables.keys())
        if missing_vars:
            raise ValidationError(
                f"Missing required template variables: {missing_vars}"
            )
        
        # Validate variable values
        for var_name, var_value in variables.items():
            if not isinstance(var_value, str):
                raise ValidationError(f"Variable '{var_name}' must be a string")
            
            if len(var_value) > 10000:
                raise ValidationError(f"Variable '{var_name}' exceeds maximum length")
            
            # Security check: prevent injection in variable values
            if re.search(r'<script[^>]*>', var_value, re.IGNORECASE):
                raise SecurityError(f"Variable '{var_name}' contains potentially malicious content")
        
        return True
    
    def render(self, variables: Dict[str, str]) -> Dict[str, str]:
        """
        Render the email template with provided variables.
        
        Args:
            variables: Dictionary of variable names to values
            
        Returns:
            Dictionary with 'subject' and 'body' keys
            
        Raises:
            ValidationError: If required variables are missing
            EmailTemplateError: If rendering fails
            SecurityError: If rendering produces malicious content
        """
        try:
            self.validate_variables(variables)
            
            rendered_subject = self.subject
            rendered_body = self.body
            
            # Sort variables by length (longest first) to prevent partial replacements
            sorted_vars = sorted(variables.items(), key=lambda x: len(x[0]), reverse=True)
            
            for var_name, var_value in sorted_vars:
                placeholder = f'{{{{{var_name}}}}}'
                # Escape special regex characters in variable name
                escaped_placeholder = re.escape(placeholder)
                rendered_subject = re.sub(escaped_placeholder, str(var_value), rendered_subject)
                rendered_body = re.sub(escaped_placeholder, str(var_value), rendered_body)
            
            # Final security check on rendered content
            if re.search(r'<script[^>]*>', rendered_body, re.IGNORECASE):
                raise SecurityError("Rendered template contains potentially malicious content")
            
            return {
                'subject': rendered_subject,
                'body': rendered_body
            }
        except (ValidationError, SecurityError):
            raise
        except Exception as e:
            logger.error(f"Failed to render email template: {e}")
            raise EmailTemplateError(f"Template rendering failed: {e}") from e
    
    def to_dict(self) -> Dict[str, Any]:
        """
        Convert template to dictionary for serialization.
        
        Returns:
            Dictionary representation of the template
        """
        return {
            'subject': self.subject,
            'body': self.body,
            'template_variables': list(self.template_variables),
            'version': self.version,
            'created_at': self.created_at.isoformat()
        }
    
    @classmethod
    def from_dict(cls, data: Dict[str, Any]) -> 'EmailTemplate':
        """
        Create template from dictionary.
        
        Args:
            data: Dictionary containing template data
            
        Returns:
            EmailTemplate instance
            
        Raises:
            ValidationError: If dictionary data is invalid
        """
        try:
            required_fields = ['subject', 'body']
            for field in required_fields:
                if field not in data:
                    raise ValidationError(f"Missing required field: {field}")
            
            return cls(
                subject=data['subject'],
                body=data['body'],
                version=data.get('version', datetime.now().isoformat()),
                created_at=datetime.fromisoformat(data.get('created_at', datetime.now().isoformat()))
            )
        except Exception as e:
            logger.error(f"Failed to create template from dictionary: {e}")
            raise EmailTemplateError(f"Template creation from dict failed: {e}") from e


class BaseEmailSequence(ABC):
    """
    Abstract base class for email sequences.
    
    Provides common functionality for managing email sequences,
    including template management, validation, and sending.
    """
    
    def __init__(self, sequence_name: str, trigger: EmailTrigger):
        """
        Initialize the email sequence.
        
        Args:
            sequence_name: Name of the email sequence
            trigger: Trigger event for the sequence
            
        Raises:
            ValidationError: If parameters are invalid
        """
        if not sequence_name or not isinstance(sequence_name, str):
            raise ValidationError("Sequence name must be a non-empty string")
        
        if not isinstance(trigger, EmailTrigger):
            raise ValidationError("Trigger must be an EmailTrigger enum value")
        
        self.sequence_name = sequence_name
        self.trigger = trigger
        self.templates: List[EmailTemplate] = []
        self.logger = logging.getLogger(f"{__name__}.{self.__class__.__name__}")
        self._send_count: int = 0
        self._error_count: int = 0
    
    @abstractmethod
    def get_templates(self) -> List[EmailTemplate]:
        """Get the list of email templates for this sequence."""
        pass
    
    def add_template(self, template: EmailTemplate) -> None:
        """
        Add a template to the sequence.
        
        Args:
            template: EmailTemplate instance to add
            
        Raises:
            ValueError: If template is None
            ValidationError: If template is invalid
        """
        if template is None:
            raise ValueError("Template cannot be None")
        
        if not isinstance(template, EmailTemplate):
            raise ValidationError("Template must be an EmailTemplate instance")
        
        # Check for duplicate templates
        template_hash = hashlib.md5(
            f"{template.subject}{template.body}".encode()
        ).hexdigest()
        
        for existing_template in self.templates:
            existing_hash = hashlib.md5(
                f"{existing_template.subject}{existing_template.body}".encode()
            ).hexdigest()
            if template_hash == existing_hash:
                self.logger.warning(f"Duplicate template detected in sequence {self.sequence_name}")
                return
        
        self.templates.append(template)
        self.logger.debug(f"Added template to sequence {self.sequence_name}")
    
    def remove_template(self, template: EmailTemplate) -> bool:
        """
        Remove a template from the sequence.
        
        Args:
            template: EmailTemplate instance to remove
            
        Returns:
            True if template was removed, False if not found
        """
        try:
            self.templates.remove(template)
            self.logger.debug(f"Removed template from sequence {self.sequence_name}")
            return True
        except ValueError:
            self.logger.warning(f"Template not found in sequence {self.sequence_name}")
            return False
    
    def validate_sequence(self) -> bool:
        """
        Validate the entire email sequence.
        
        Returns:
            True if sequence is valid
            
        Raises:
            EmailTemplateError: If sequence validation fails
        """
        if not self.templates:
            raise EmailTemplateError(f"Sequence {self.sequence_name} has no templates")
        
        for i, template in enumerate(self.templates):
            if not isinstance(template, EmailTemplate):
                raise EmailTemplateError(
                    f"Template at index {i} is not an EmailTemplate instance"
                )
        
        self.logger.info(f"Sequence {self.sequence_name} validated successfully")
        return True
    
    def send_email(self, template: EmailTemplate, variables: Dict[str, str]) -> bool:
        """
        Send an email using the given template and variables.
        
        Args:
            template: EmailTemplate to use
            variables: Variables for template rendering
            
        Returns:
            True if email was sent successfully
            
        Raises:
            EmailTemplateError: If sending fails
            ValidationError: If parameters are invalid
        """
        if not isinstance(template, EmailTemplate):
            raise ValidationError("Template must be an EmailTemplate instance")
        
        if not isinstance(variables, dict):
            raise ValidationError("Variables must be a dictionary")
        
        try:
            rendered = template.render(variables)
            
            # Validate rendered content
            if not rendered.get('subject') or not rendered.get('body'):
                raise EmailTemplateError("Rendered template has empty subject or body")
            
            # In production, this would call an email service
            self.logger.info(
                f"Sending email with subject: {rendered['subject'][:50]}..."
            )
            
            # Simulate successful send with rate limiting
            self._send_count += 1
            self.logger.debug(f"Email sent successfully. Total sent: {self._send_count}")
            
            return True
        except (ValidationError, SecurityError):
            raise
        except Exception as e:
            self._error_count += 1
            self.logger.error(f"Failed to send email (error #{self._error_count}): {e}")
            raise EmailTemplateError(f"Email sending failed: {e}") from e
    
    def send_batch(self, templates: List[EmailTemplate], 
                   variables_list: List[Dict[str, str]]) -> List[bool]:
        """
        Send multiple emails in batch.
        
        Args:
            templates: List of EmailTemplate instances
            variables_list: List of variable dictionaries
            
        Returns:
            List of boolean results for each email
            
        Raises:
            ValidationError: If input lists have different lengths
        """
        if len(templates) != len(variables_list):
            raise ValidationError(
                "Templates and variables lists must have the same length"
            )
        
        results: List[bool] = []
        for template, variables in zip(templates, variables_list):
            try:
                result = self.send_email(template, variables)
                results.append(result)
            except Exception as e:
                self.logger.error(f"Batch send failed for template: {e}")
                results.append(False)
        
        return results
    
    def get_statistics(self) -> Dict[str, Any]:
        """
        Get email sequence statistics.
        
        Returns:
            Dictionary with sequence statistics
        """
        return {
            'sequence_name': self.sequence_name,
            'trigger': self.trigger.value,
            'template_count': len(self.templates),
            'send_count': self._send_count,
            'error_count': self._error_count,
            'success_rate': (self._send_count / (self._send_count + self._error_count) * 100 
                           if (self._send_count + self._error_count) > 0 else 0)
        }


class InternalStaffOnboardingSequence(BaseEmailSequence):
    """
    Email sequence for internal staff onboarding to the credit system.
    
    This sequence is triggered when staff members are granted permissions
    to manage credits and redeemable codes.
    """
    
    def __init__(self):
        """Initialize the internal staff onboarding sequence."""
        try:
            super().__init__(
                sequence_name="internal_staff_onboarding",
                trigger=EmailTrigger.STAFF_GRANTED_PERMISSIONS
            )
            self._initialize_templates()
            self.logger.info("InternalStaffOnboardingSequence initialized successfully")
        except Exception as e:
            self.logger.error(f"Failed to initialize InternalStaffOnboardingSequence: {e}")
            raise
    
    def _initialize_templates(self) -> None:
        """Initialize the email templates for this sequence."""
        try:
            # Welcome template
            welcome_template = EmailTemplate(
                subject="Welcome to the Credit Ledger System - {{staff_name}}",
                body="""
Hello {{staff_name}},

You have been granted access to the Credit Ledger System. As a staff member,
you can now manage credits and redeemable codes for our customers.

Key Features:
- Grant credits to customer accounts
- Create redeemable codes for promotions
- View credit usage and expiration
- Generate reports on credit distribution

To get started, please visit: {{portal_url}}

Your permissions include:
- Role: {{staff_role}}
- Access Level: {{access_level}}
- Expiration: {{permission_expiry}}

If you have any questions, please contact your administrator.

Best regards,
The Datum Team
                """,
                version="1.0.0"
            )
            self.add_template(welcome_template)
            
            # Training template
            training_template = EmailTemplate(
                subject="Credit Ledger Training Resources - {{staff_name}}",
                body="""
Hello {{staff_name}},

To help you get started with the Credit Ledger System, we've prepared
some training resources:

1. Quick Start Guide: {{quick_start_url}}
2. Video Tutorial: {{video_tutorial_url}}
3. API Documentation: {{api_docs_url}}
4. Best Practices: {{best_practices_url}}

Please complete the training within {{training_days}} days.

Your training progress can be tracked at: {{training_tracker_url}}

Best regards,
The Datum Team
                """,
                version="1.0.0"
            )
            self.add_template(training_template)
            
            self.logger.info(f"Initialized {len(self.templates)} templates for {self.sequence_name}")
        except Exception as e:
            self.logger.error(f"Failed to initialize templates: {e}")
            raise EmailTemplateError(f"Template initialization failed: {e}") from e
    
    def get_templates(self) -> List[EmailTemplate]:
        """Get the list of email templates for this sequence."""
        return self.templates.copy()
    
    def send_onboarding_emails(self, staff_info: Dict[str, str]) -> bool:
        """
        Send complete onboarding email sequence to a staff member.
        
        Args:
            staff_info: Dictionary containing staff member information
            
        Returns:
            True if all emails were sent successfully
            
        Raises:
            ValidationError: If staff_info is invalid
            EmailTemplateError: If email sending fails
        """
        required_fields = ['staff_name', 'staff_role', 'access_level', 'portal_url']
        missing_fields = [field for field in required_fields if field not in staff_info]
        
        if
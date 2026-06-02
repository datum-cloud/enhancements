package main

import (
	"context"
	"crypto/rand"
	"database/sql"
	"encoding/hex"
	"errors"
	"fmt"
	"math/big"
	"net/http"
	"regexp"
	"strings"
	"sync"
	"time"

	"github.com/go-playground/validator/v10"
	"github.com/google/uuid"
	"github.com/gorilla/mux"
	"github.com/lib/pq"
	"github.com/rs/zerolog"
	"github.com/rs/zerolog/hlog"
	"github.com/rs/zerolog/log"
	"golang.org/x/crypto/bcrypt"
)

// ============================================================================
// Custom Errors
// ============================================================================

var (
	ErrCreditGrantNotFound      = errors.New("credit grant not found")
	ErrRedeemableCodeNotFound   = errors.New("redeemable code not found")
	ErrInvoiceNotFound          = errors.New("invoice not found")
	ErrInsufficientCredit       = errors.New("insufficient credit remaining")
	ErrCodeExpired              = errors.New("redeemable code has expired")
	ErrCodeMaxRedemptions       = errors.New("redeemable code has reached maximum redemptions")
	ErrCodeInactive             = errors.New("redeemable code is inactive")
	ErrInvalidAmount            = errors.New("invalid amount: must be greater than zero")
	ErrInvalidBillingAccount    = errors.New("invalid billing account ID")
	ErrDuplicateCode            = errors.New("redeemable code already exists")
	ErrConcurrentUpdate         = errors.New("concurrent update detected")
	ErrDatabaseOperation        = errors.New("database operation failed")
	ErrValidationFailed         = errors.New("validation failed")
	ErrUnauthorized             = errors.New("unauthorized operation")
	ErrInternalServer           = errors.New("internal server error")
)

// ============================================================================
// Domain Types
// ============================================================================

// CreditGrant represents a prepaid credit applied to a billing account.
type CreditGrant struct {
	ID               string    `json:"id" db:"id"`
	BillingAccountID string    `json:"billing_account_id" db:"billing_account_id" validate:"required,uuid"`
	AmountCents      int64     `json:"amount_cents" db:"amount_cents" validate:"required,gt=0"`
	RemainingCents   int64     `json:"remaining_cents" db:"remaining_cents" validate:"gte=0"`
	Note             string    `json:"note" db:"note" validate:"max=500"`
	ExpiresAt        *time.Time `json:"expires_at,omitempty" db:"expires_at"`
	CreatedBy        string    `json:"created_by" db:"created_by" validate:"required"`
	CreatedAt        time.Time `json:"created_at" db:"created_at"`
	UpdatedAt        time.Time `json:"updated_at" db:"updated_at"`
}

// RedeemableCode represents a code that can be redeemed for credit.
type RedeemableCode struct {
	Code               string    `json:"code" db:"code" validate:"required,min=6,max=32"`
	AmountCents        int64     `json:"amount_cents" db:"amount_cents" validate:"required,gt=0"`
	MaxRedemptions     int       `json:"max_redemptions" db:"max_redemptions" validate:"required,gt=0"`
	CurrentRedemptions int       `json:"current_redemptions" db:"current_redemptions" validate:"gte=0"`
	ExpiresAt          *time.Time `json:"expires_at,omitempty" db:"expires_at"`
	IsActive           bool      `json:"is_active" db:"is_active"`
	CreatedBy          string    `json:"created_by" db:"created_by" validate:"required"`
	CreatedAt          time.Time `json:"created_at" db:"created_at"`
	UpdatedAt          time.Time `json:"updated_at" db:"updated_at"`
}

// Invoice represents a billing invoice with credit application.
type Invoice struct {
	ID                 string    `json:"id" db:"id"`
	BillingAccountID   string    `json:"billing_account_id" db:"billing_account_id" validate:"required,uuid"`
	TotalCents         int64     `json:"total_cents" db:"total_cents" validate:"required,gt=0"`
	CreditAppliedCents int64     `json:"credit_applied_cents" db:"credit_applied_cents" validate:"gte=0"`
	ChargeCents        int64     `json:"charge_cents" db:"charge_cents" validate:"gte=0"`
	Status             string    `json:"status" db:"status" validate:"required,oneof=pending paid failed"`
	StripePaymentIntentID *string `json:"stripe_payment_intent_id,omitempty" db:"stripe_payment_intent_id"`
	CreatedAt          time.Time `json:"created_at" db:"created_at"`
	UpdatedAt          time.Time `json:"updated_at" db:"updated_at"`
}

// ============================================================================
// Repository Interfaces
// ============================================================================

// CreditGrantRepository defines the interface for credit grant persistence.
type CreditGrantRepository interface {
	Create(ctx context.Context, grant *CreditGrant) error
	GetByID(ctx context.Context, id string) (*CreditGrant, error)
	GetByBillingAccount(ctx context.Context, billingAccountID string) ([]*CreditGrant, error)
	UpdateRemaining(ctx context.Context, id string, remainingCents int64) error
	GetExpiredGrants(ctx context.Context) ([]*CreditGrant, error)
}

// RedeemableCodeRepository defines the interface for redeemable code persistence.
type RedeemableCodeRepository interface {
	Create(ctx context.Context, code *RedeemableCode) error
	GetByCode(ctx context.Context, code string) (*RedeemableCode, error)
	IncrementRedemptions(ctx context.Context, code string) error
	GetActiveCodes(ctx context.Context) ([]*RedeemableCode, error)
}

// InvoiceRepository defines the interface for invoice persistence.
type InvoiceRepository interface {
	Create(ctx context.Context, invoice *Invoice) error
	GetByID(ctx context.Context, id string) (*Invoice, error)
	GetByBillingAccount(ctx context.Context, billingAccountID string) ([]*Invoice, error)
	UpdateStatus(ctx context.Context, id string, status string, stripePaymentIntentID *string) error
}

// ============================================================================
// PostgreSQL Implementations
// ============================================================================

type postgresCreditGrantRepository struct {
	db *sql.DB
}

// NewPostgresCreditGrantRepository creates a new PostgreSQL credit grant repository.
func NewPostgresCreditGrantRepository(db *sql.DB) CreditGrantRepository {
	return &postgresCreditGrantRepository{db: db}
}

// Create inserts a new credit grant into the database.
func (r *postgresCreditGrantRepository) Create(ctx context.Context, grant *CreditGrant) error {
	const query = `
		INSERT INTO credit_grants (id, billing_account_id, amount_cents, remaining_cents, note, expires_at, created_by, created_at, updated_at)
		VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
	`

	_, err := r.db.ExecContext(ctx, query,
		grant.ID, grant.BillingAccountID, grant.AmountCents, grant.RemainingCents,
		grant.Note, grant.ExpiresAt, grant.CreatedBy, grant.CreatedAt, grant.UpdatedAt,
	)
	if err != nil {
		log.Error().Err(err).Str("grant_id", grant.ID).Msg("failed to create credit grant")
		return fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}
	return nil
}

// GetByID retrieves a credit grant by its ID.
func (r *postgresCreditGrantRepository) GetByID(ctx context.Context, id string) (*CreditGrant, error) {
	const query = `
		SELECT id, billing_account_id, amount_cents, remaining_cents, note, expires_at, created_by, created_at, updated_at 
		FROM credit_grants WHERE id = $1
	`

	grant := &CreditGrant{}
	err := r.db.QueryRowContext(ctx, query, id).Scan(
		&grant.ID, &grant.BillingAccountID, &grant.AmountCents, &grant.RemainingCents,
		&grant.Note, &grant.ExpiresAt, &grant.CreatedBy, &grant.CreatedAt, &grant.UpdatedAt,
	)
	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, fmt.Errorf("%w: %s", ErrCreditGrantNotFound, id)
		}
		log.Error().Err(err).Str("grant_id", id).Msg("failed to get credit grant")
		return nil, fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}
	return grant, nil
}

// GetByBillingAccount retrieves all active credit grants for a billing account.
func (r *postgresCreditGrantRepository) GetByBillingAccount(ctx context.Context, billingAccountID string) ([]*CreditGrant, error) {
	const query = `
		SELECT id, billing_account_id, amount_cents, remaining_cents, note, expires_at, created_by, created_at, updated_at 
		FROM credit_grants 
		WHERE billing_account_id = $1 
		AND (expires_at IS NULL OR expires_at > NOW()) 
		AND remaining_cents > 0 
		ORDER BY created_at DESC
	`

	rows, err := r.db.QueryContext(ctx, query, billingAccountID)
	if err != nil {
		log.Error().Err(err).Str("billing_account_id", billingAccountID).Msg("failed to query credit grants")
		return nil, fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}
	defer rows.Close()

	var grants []*CreditGrant
	for rows.Next() {
		grant := &CreditGrant{}
		if err := rows.Scan(
			&grant.ID, &grant.BillingAccountID, &grant.AmountCents, &grant.RemainingCents,
			&grant.Note, &grant.ExpiresAt, &grant.CreatedBy, &grant.CreatedAt, &grant.UpdatedAt,
		); err != nil {
			log.Error().Err(err).Msg("failed to scan credit grant row")
			return nil, fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
		}
		grants = append(grants, grant)
	}

	if err := rows.Err(); err != nil {
		log.Error().Err(err).Msg("error iterating credit grant rows")
		return nil, fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}

	return grants, nil
}

// UpdateRemaining updates the remaining cents of a credit grant with optimistic locking.
func (r *postgresCreditGrantRepository) UpdateRemaining(ctx context.Context, id string, remainingCents int64) error {
	const query = `
		UPDATE credit_grants 
		SET remaining_cents = $1, updated_at = NOW() 
		WHERE id = $2 AND remaining_cents >= $1
	`

	result, err := r.db.ExecContext(ctx, query, remainingCents, id)
	if err != nil {
		log.Error().Err(err).Str("grant_id", id).Int64("remaining", remainingCents).Msg("failed to update credit grant remaining")
		return fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}

	rowsAffected, err := result.RowsAffected()
	if err != nil {
		log.Error().Err(err).Msg("failed to get rows affected")
		return fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}

	if rowsAffected == 0 {
		return fmt.Errorf("%w: %s", ErrCreditGrantNotFound, id)
	}
	return nil
}

// GetExpiredGrants retrieves all expired credit grants that still have remaining balance.
func (r *postgresCreditGrantRepository) GetExpiredGrants(ctx context.Context) ([]*CreditGrant, error) {
	const query = `
		SELECT id, billing_account_id, amount_cents, remaining_cents, note, expires_at, created_by, created_at, updated_at 
		FROM credit_grants 
		WHERE expires_at IS NOT NULL AND expires_at <= NOW() AND remaining_cents > 0
	`

	rows, err := r.db.QueryContext(ctx, query)
	if err != nil {
		log.Error().Err(err).Msg("failed to query expired credit grants")
		return nil, fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}
	defer rows.Close()

	var grants []*CreditGrant
	for rows.Next() {
		grant := &CreditGrant{}
		if err := rows.Scan(
			&grant.ID, &grant.BillingAccountID, &grant.AmountCents, &grant.RemainingCents,
			&grant.Note, &grant.ExpiresAt, &grant.CreatedBy, &grant.CreatedAt, &grant.UpdatedAt,
		); err != nil {
			log.Error().Err(err).Msg("failed to scan expired credit grant row")
			return nil, fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
		}
		grants = append(grants, grant)
	}

	if err := rows.Err(); err != nil {
		log.Error().Err(err).Msg("error iterating expired credit grant rows")
		return nil, fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}

	return grants, nil
}

// ============================================================================
// Service Layer
// ============================================================================

// CreditService handles credit grant operations.
type CreditService struct {
	grantRepo CreditGrantRepository
	codeRepo  RedeemableCodeRepository
	invRepo   InvoiceRepository
	validator *validator.Validate
}

// NewCreditService creates a new credit service with all dependencies.
func NewCreditService(
	grantRepo CreditGrantRepository,
	codeRepo RedeemableCodeRepository,
	invRepo InvoiceRepository,
) *CreditService {
	return &CreditService{
		grantRepo: grantRepo,
		codeRepo:  codeRepo,
		invRepo:   invRepo,
		validator: validator.New(),
	}
}

// CreateCreditGrant creates a new credit grant for a billing account.
func (s *CreditService) CreateCreditGrant(ctx context.Context, billingAccountID string, amountCents int64, note string, expiresAt *time.Time, createdBy string) (*CreditGrant, error) {
	// Input validation
	if err := s.validateCreditGrantInput(billingAccountID, amountCents, note); err != nil {
		return nil, err
	}

	// Create grant
	grant := &CreditGrant{
		ID:               uuid.New().String(),
		BillingAccountID: billingAccountID,
		AmountCents:      amountCents,
		RemainingCents:   amountCents,
		Note:             sanitizeNote(note),
		ExpiresAt:        expiresAt,
		CreatedBy:        createdBy,
		CreatedAt:        time.Now().UTC(),
		UpdatedAt:        time.Now().UTC(),
	}

	if err := s.grantRepo.Create(ctx, grant); err != nil {
		log.Error().Err(err).Str("billing_account_id", billingAccountID).Int64("amount", amountCents).Msg("failed to create credit grant")
		return nil, fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}

	log.Info().
		Str("grant_id", grant.ID).
		Str("billing_account_id", billingAccountID).
		Int64("amount_cents", amountCents).
		Str("created_by", createdBy).
		Msg("credit grant created successfully")

	return grant, nil
}

// RedeemCode redeems a code and applies credit to the billing account.
func (s *CreditService) RedeemCode(ctx context.Context, code string, billingAccountID string) (*CreditGrant, error) {
	// Input validation
	if err := s.validateRedeemCodeInput(code, billingAccountID); err != nil {
		return nil, err
	}

	// Get code
	redeemableCode, err := s.codeRepo.GetByCode(ctx, strings.ToUpper(code))
	if err != nil {
		if errors.Is(err, ErrRedeemableCodeNotFound) {
			return nil, fmt.Errorf("%w: %s", ErrRedeemableCodeNotFound, code)
		}
		return nil, fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}

	// Validate code
	if err := s.validateRedeemableCode(redeemableCode); err != nil {
		return nil, err
	}

	// Create credit grant
	grant, err := s.CreateCreditGrant(ctx, billingAccountID, redeemableCode.AmountCents, fmt.Sprintf("Redeemed code: %s", redeemableCode.Code), nil, "system")
	if err != nil {
		return nil, err
	}

	// Increment redemptions
	if err := s.codeRepo.IncrementRedemptions(ctx, redeemableCode.Code); err != nil {
		log.Error().Err(err).Str("code", redeemableCode.Code).Msg("failed to increment redemptions")
		// Rollback credit grant creation
		if rollbackErr := s.grantRepo.UpdateRemaining(ctx, grant.ID, 0); rollbackErr != nil {
			log.Error().Err(rollbackErr).Str("grant_id", grant.ID).Msg("failed to rollback credit grant")
		}
		return nil, fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}

	log.Info().
		Str("code", redeemableCode.Code).
		Str("billing_account_id", billingAccountID).
		Int64("amount_cents", redeemableCode.AmountCents).
		Msg("code redeemed successfully")

	return grant, nil
}

// ApplyCreditsToInvoice applies available credits to an invoice.
func (s *CreditService) ApplyCreditsToInvoice(ctx context.Context, invoiceID string) (*Invoice, error) {
	// Get invoice
	invoice, err := s.invRepo.GetByID(ctx, invoiceID)
	if err != nil {
		return nil, fmt.Errorf("%w: %v", ErrInvoiceNotFound, err)
	}

	// Get available credits
	grants, err := s.grantRepo.GetByBillingAccount(ctx, invoice.BillingAccountID)
	if err != nil {
		return nil, fmt.Errorf("%w: %v", ErrDatabaseOperation, err)
	}

	// Calculate credit application
	remainingCharge := invoice.TotalCents
	credit
package main

import (
    "context"
    "crypto/rand"
    "encoding/hex"
    "errors"
    "fmt"
    "math"
    "strings"
    "sync"
    "time"

    "github.com/amberflo/amberflo-go"
    "github.com/google/uuid"
    "go.uber.org/zap"
)

// Custom error types for credit ledger operations
var (
    ErrAccountNotFound          = errors.New("billing account not found")
    ErrInvalidAmount            = errors.New("credit amount must be greater than zero")
    ErrInvalidExpiry            = errors.New("expiry date must be in the future")
    ErrCodeNotFound             = errors.New("redemption code not found")
    ErrCodeExpired              = errors.New("redemption code has expired")
    ErrCodeAlreadyRedeemed      = errors.New("redemption code already used")
    ErrCodeMaxRedemptions       = errors.New("redemption code has reached maximum uses")
    ErrInsufficientCredit       = errors.New("insufficient credit balance")
    ErrInvalidAccountID         = errors.New("invalid billing account ID format")
    ErrInvalidNoteLength        = errors.New("note must be between 1 and 500 characters")
    ErrConcurrentModification   = errors.New("concurrent modification detected, retry operation")
    ErrInvalidAmountPrecision   = errors.New("amount must have at most 2 decimal places")
    ErrAmountExceedsMax         = errors.New("amount exceeds maximum allowed value")
    ErrInvalidCodeFormat        = errors.New("redemption code format is invalid")
    ErrCodeAlreadyExists        = errors.New("redemption code already exists")
    ErrGrantNotFound            = errors.New("credit grant not found")
    ErrGrantExpired             = errors.New("credit grant has expired")
    ErrGrantFullyConsumed       = errors.New("credit grant has been fully consumed")
    ErrInvalidCreatedBy         = errors.New("created by identifier is invalid")
)

const (
    maxAmount          = 1000000.00
    maxNoteLength      = 500
    minNoteLength      = 1
    maxAccountIDLength = 100
    maxRetries         = 3
    retryBackoff       = 100 * time.Millisecond
    maxCodeLength      = 64
    codeBytesLength    = 16
)

// CreditGrant represents a credit grant applied to a billing account
type CreditGrant struct {
    ID        string     `json:"id"`
    AccountID string     `json:"account_id"`
    Amount    float64    `json:"amount"`
    Remaining float64    `json:"remaining"`
    Note      string     `json:"note"`
    ExpiresAt *time.Time `json:"expires_at,omitempty"`
    CreatedBy string     `json:"created_by"`
    CreatedAt time.Time  `json:"created_at"`
    UpdatedAt time.Time  `json:"updated_at"`
    Version   int        `json:"version"`
}

// RedemptionCode represents a redeemable code for credit
type RedemptionCode struct {
    Code               string     `json:"code"`
    Amount             float64    `json:"amount"`
    MaxRedemptions     int        `json:"max_redemptions"`
    CurrentRedemptions int        `json:"current_redemptions"`
    ExpiresAt          *time.Time `json:"expires_at,omitempty"`
    CreatedBy          string     `json:"created_by"`
    CreatedAt          time.Time  `json:"created_at"`
    IsActive           bool       `json:"is_active"`
}

// CreditLedger manages all credit operations
type CreditLedger struct {
    mu             sync.RWMutex
    grants         map[string]*CreditGrant
    codes          map[string]*RedemptionCode
    accountCredits map[string][]string // accountID -> grantIDs
    logger         *zap.Logger
    amberfloClient *amberflo.Client
    maxRetries     int
    retryBackoff   time.Duration
}

// NewCreditLedger creates a new CreditLedger instance
func NewCreditLedger(amberfloClient *amberflo.Client, logger *zap.Logger) *CreditLedger {
    if logger == nil {
        var err error
        logger, err = zap.NewProduction()
        if err != nil {
            logger = zap.NewNop()
        }
    }

    return &CreditLedger{
        grants:         make(map[string]*CreditGrant),
        codes:          make(map[string]*RedemptionCode),
        accountCredits: make(map[string][]string),
        logger:         logger,
        amberfloClient: amberfloClient,
        maxRetries:     maxRetries,
        retryBackoff:   retryBackoff,
    }
}

// validateAccountID validates the format of a billing account ID
func validateAccountID(accountID string) error {
    trimmedID := strings.TrimSpace(accountID)
    if trimmedID == "" {
        return fmt.Errorf("%w: account ID cannot be empty", ErrInvalidAccountID)
    }
    if len(trimmedID) > maxAccountIDLength {
        return fmt.Errorf("%w: account ID too long (max %d characters)", ErrInvalidAccountID, maxAccountIDLength)
    }
    if _, err := uuid.Parse(trimmedID); err != nil {
        return fmt.Errorf("%w: must be a valid UUID, got: %s", ErrInvalidAccountID, trimmedID)
    }
    return nil
}

// validateAmount validates the credit amount
func validateAmount(amount float64) error {
    if amount <= 0 {
        return ErrInvalidAmount
    }
    if amount > maxAmount {
        return fmt.Errorf("%w: amount %f exceeds maximum of $%.2f", ErrAmountExceedsMax, amount, maxAmount)
    }
    // Check for more than 2 decimal places
    rounded := math.Round(amount*100) / 100
    if math.Abs(rounded-amount) > 0.001 {
        return ErrInvalidAmountPrecision
    }
    return nil
}

// validateExpiry validates the expiry date
func validateExpiry(expiresAt *time.Time) error {
    if expiresAt != nil && expiresAt.Before(time.Now()) {
        return ErrInvalidExpiry
    }
    return nil
}

// validateNote validates the note text
func validateNote(note string) error {
    trimmedNote := strings.TrimSpace(note)
    if len(trimmedNote) < minNoteLength || len(trimmedNote) > maxNoteLength {
        return fmt.Errorf("%w: note length must be between %d and %d characters", 
            ErrInvalidNoteLength, minNoteLength, maxNoteLength)
    }
    // Sanitize input - remove potentially dangerous characters
    sanitized := strings.Map(func(r rune) rune {
        if r < 32 || r > 126 {
            return -1
        }
        return r
    }, trimmedNote)
    if sanitized != trimmedNote {
        return fmt.Errorf("%w: note contains invalid characters", ErrInvalidNoteLength)
    }
    return nil
}

// validateCreatedBy validates the creator identifier
func validateCreatedBy(createdBy string) error {
    trimmedCreator := strings.TrimSpace(createdBy)
    if trimmedCreator == "" {
        return ErrInvalidCreatedBy
    }
    if len(trimmedCreator) > 100 {
        return fmt.Errorf("%w: creator identifier too long (max 100 characters)", ErrInvalidCreatedBy)
    }
    return nil
}

// generateCode generates a cryptographically secure random code
func generateCode() (string, error) {
    bytes := make([]byte, codeBytesLength)
    if _, err := rand.Read(bytes); err != nil {
        return "", fmt.Errorf("failed to generate random code: %w", err)
    }
    return hex.EncodeToString(bytes), nil
}

// roundAmount rounds a float64 to 2 decimal places
func roundAmount(amount float64) float64 {
    return math.Round(amount*100) / 100
}

// ApplyCredit applies a credit grant to a billing account
func (cl *CreditLedger) ApplyCredit(ctx context.Context, accountID string, amount float64, note string, expiresAt *time.Time, createdBy string) (*CreditGrant, error) {
    // Input validation
    if err := validateAccountID(accountID); err != nil {
        cl.logger.Warn("invalid account ID", 
            zap.String("accountID", accountID), 
            zap.Error(err))
        return nil, err
    }
    if err := validateAmount(amount); err != nil {
        cl.logger.Warn("invalid amount", 
            zap.Float64("amount", amount), 
            zap.Error(err))
        return nil, err
    }
    if err := validateNote(note); err != nil {
        cl.logger.Warn("invalid note", 
            zap.String("note", note), 
            zap.Error(err))
        return nil, err
    }
    if err := validateExpiry(expiresAt); err != nil {
        cl.logger.Warn("invalid expiry", 
            zap.Time("expiresAt", *expiresAt), 
            zap.Error(err))
        return nil, err
    }
    if err := validateCreatedBy(createdBy); err != nil {
        cl.logger.Warn("invalid created by", 
            zap.String("createdBy", createdBy), 
            zap.Error(err))
        return nil, err
    }

    // Retry logic for concurrent modification
    var grant *CreditGrant
    var err error
    for i := 0; i < cl.maxRetries; i++ {
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        default:
        }

        grant, err = cl.applyCreditWithRetry(ctx, accountID, amount, note, expiresAt, createdBy)
        if err == nil || !errors.Is(err, ErrConcurrentModification) {
            break
        }
        cl.logger.Warn("retrying credit application due to concurrent modification",
            zap.Int("attempt", i+1),
            zap.String("accountID", accountID),
            zap.Int("maxRetries", cl.maxRetries))
        
        backoffDuration := cl.retryBackoff * time.Duration(i+1)
        select {
        case <-ctx.Done():
            return nil, ctx.Err()
        case <-time.After(backoffDuration):
        }
    }

    if err != nil {
        cl.logger.Error("failed to apply credit after retries",
            zap.String("accountID", accountID),
            zap.Float64("amount", amount),
            zap.Int("attempts", cl.maxRetries),
            zap.Error(err))
        return nil, err
    }

    // Record in Amberflo asynchronously
    go func(grant *CreditGrant) {
        if err := cl.recordAmberfloCredit(context.Background(), grant); err != nil {
            cl.logger.Error("failed to record credit in Amberflo",
                zap.String("grantID", grant.ID),
                zap.String("accountID", grant.AccountID),
                zap.Float64("amount", grant.Amount),
                zap.Error(err))
        }
    }(grant)

    cl.logger.Info("credit applied successfully",
        zap.String("grantID", grant.ID),
        zap.String("accountID", accountID),
        zap.Float64("amount", amount),
        zap.Float64("remaining", grant.Remaining),
        zap.String("createdBy", createdBy))

    return grant, nil
}

// applyCreditWithRetry applies credit with optimistic locking
func (cl *CreditLedger) applyCreditWithRetry(ctx context.Context, accountID string, amount float64, note string, expiresAt *time.Time, createdBy string) (*CreditGrant, error) {
    cl.mu.Lock()
    defer cl.mu.Unlock()

    // Check for concurrent modification using version
    existingGrants := cl.accountCredits[accountID]
    for _, grantID := range existingGrants {
        if grant, exists := cl.grants[grantID]; exists {
            if grant.Version != grant.Version {
                return nil, ErrConcurrentModification
            }
        }
    }

    // Create new credit grant
    grant := &CreditGrant{
        ID:        uuid.New().String(),
        AccountID: accountID,
        Amount:    roundAmount(amount),
        Remaining: roundAmount(amount),
        Note:      strings.TrimSpace(note),
        ExpiresAt: expiresAt,
        CreatedBy: strings.TrimSpace(createdBy),
        CreatedAt: time.Now().UTC(),
        UpdatedAt: time.Now().UTC(),
        Version:   1,
    }

    // Store the grant
    cl.grants[grant.ID] = grant
    cl.accountCredits[accountID] = append(cl.accountCredits[accountID], grant.ID)

    return grant, nil
}

// RedeemCode redeems a credit code for a billing account
func (cl *CreditLedger) RedeemCode(ctx context.Context, code string, accountID string) (*CreditGrant, error) {
    // Input validation
    if err := validateAccountID(accountID); err != nil {
        cl.logger.Warn("invalid account ID for code redemption",
            zap.String("accountID", accountID),
            zap.Error(err))
        return nil, err
    }

    trimmedCode := strings.TrimSpace(code)
    if trimmedCode == "" || len(trimmedCode) > maxCodeLength {
        return nil, ErrInvalidCodeFormat
    }

    cl.mu.Lock()
    defer cl.mu.Unlock()

    // Find the redemption code
    redemptionCode, exists := cl.codes[trimmedCode]
    if !exists {
        cl.logger.Warn("redemption code not found",
            zap.String("code", trimmedCode))
        return nil, ErrCodeNotFound
    }

    // Validate code
    if !redemptionCode.IsActive {
        return nil, ErrCodeNotFound
    }

    if redemptionCode.ExpiresAt != nil && redemptionCode.ExpiresAt.Before(time.Now()) {
        cl.logger.Warn("redemption code expired",
            zap.String("code", trimmedCode),
            zap.Time("expiresAt", *redemptionCode.ExpiresAt))
        return nil, ErrCodeExpired
    }

    if redemptionCode.CurrentRedemptions >= redemptionCode.MaxRedemptions {
        cl.logger.Warn("redemption code max uses reached",
            zap.String("code", trimmedCode),
            zap.Int("currentRedemptions", redemptionCode.CurrentRedemptions),
            zap.Int("maxRedemptions", redemptionCode.MaxRedemptions))
        return nil, ErrCodeMaxRedemptions
    }

    // Create credit grant from redemption
    grant := &CreditGrant{
        ID:        uuid.New().String(),
        AccountID: accountID,
        Amount:    roundAmount(redemptionCode.Amount),
        Remaining: roundAmount(redemptionCode.Amount),
        Note:      fmt.Sprintf("Redeemed code: %s", trimmedCode),
        CreatedBy: "system",
        CreatedAt: time.Now().UTC(),
        UpdatedAt: time.Now().UTC(),
        Version:   1,
    }

    // Update redemption code usage
    redemptionCode.CurrentRedemptions++
    if redemptionCode.CurrentRedemptions >= redemptionCode.MaxRedemptions {
        redemptionCode.IsActive = false
    }

    // Store the grant
    cl.grants[grant.ID] = grant
    cl.accountCredits[accountID] = append(cl.accountCredits[accountID], grant.ID)

    // Record in Amberflo asynchronously
    go func(grant *CreditGrant) {
        if err := cl.recordAmberfloCredit(context.Background(), grant); err != nil {
            cl.logger.Error("failed to record credit redemption in Amberflo",
                zap.String("grantID", grant.ID),
                zap.String("code", trimmedCode),
                zap.Error(err))
        }
    }(grant)

    cl.logger.Info("code redeemed successfully",
        zap.String("code", trimmedCode),
        zap.String("accountID", accountID),
        zap.Float64("amount", redemptionCode.Amount),
        zap.Int("remainingRedemptions", redemptionCode.MaxRedemptions-redemptionCode.CurrentRedemptions))

    return grant, nil
}

// CreateRedemptionCode creates a new redeemable code
func (cl *CreditLedger) CreateRedemptionCode(ctx context.Context, amount float64, maxRedemptions int, expiresAt *time.Time, createdBy string) (*RedemptionCode, error) {
    // Input validation
    if err := validateAmount(amount); err != nil {
        cl.logger.Warn("invalid amount for redemption code",
            zap.Float64("amount", amount),
            zap.Error(err))
        return nil, err
    }
    if maxRedemptions <= 0 {
        return nil, fmt.Errorf("max redemptions must be greater than zero")
    }
    if err := validateExpiry(expiresAt); err != nil {
        cl.logger.Warn("invalid expiry for redemption code",
            zap.Time("expiresAt", *expiresAt),
            zap.Error(err))
        return nil, err
    }
    if err := validateCreatedBy(createdBy); err != nil {
        cl.logger.Warn("invalid created by for redemption code",
            zap.String("createdBy", createdBy),
            zap.Error(err))
        return nil, err
    }

    // Generate unique code
    var code string
    var err error
    for attempts := 0; attempts < 10; attempts++ {
        code, err = generateCode()
        if err != nil {
            return nil, fmt.Errorf("failed to generate code: %w", err)
        }

        cl.mu.Lock()
        if _, exists := cl.codes[code]; !exists {
            break
        }
        cl.mu.Unlock()
    }

    if code == "" {
        return nil, ErrCodeAlreadyExists
    }

    redemptionCode := &RedemptionCode{
        Code:               code,
        Amount:             roundAmount(amount),
        MaxRedemptions:     maxRedemptions,
        CurrentRedemptions: 0,
        ExpiresAt:          expiresAt,
        CreatedBy:          strings.TrimSpace(createdBy),
        CreatedAt:          time.Now().UTC(),
        IsActive:           true,
    }

    cl.codes[code] = redemptionCode
    cl.mu.Unlock()

    cl.logger.Info("redemption code created",
        zap.String("code", code),
        zap.Float64("amount", amount),
        zap.Int("maxRedemptions", maxRedemptions),
        zap.String("createdBy", createdBy))

    return redemptionCode, nil
}

// GetAccountBalance gets the total available credit balance for an account
func (cl *CreditLedger) GetAccountBalance(ctx context.Context, accountID string) (
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"errors"
	"fmt"
	"log"
	"math/big"
	"strings"
	"sync"
	"time"

	"github.com/aws/aws-lambda-go/lambda"
	"github.com/aws/aws-sdk-go/aws"
	"github.com/aws/aws-sdk-go/aws/session"
	"github.com/aws/aws-sdk-go/service/dynamodb"
	"github.com/aws/aws-sdk-go/service/dynamodb/dynamodbattribute"
	"github.com/google/uuid"
	"github.com/stripe/stripe-go/v72"
	"github.com/stripe/stripe-go/v72/charge"
	"github.com/stripe/stripe-go/v72/invoice"
	"go.uber.org/zap"
)

// ============================================================================
// Domain Models
// ============================================================================

// CreditGrant represents a prepaid credit grant applied to a billing account.
type CreditGrant struct {
	ID          string    `json:"id" dynamodbav:"id"`
	AccountID   string    `json:"account_id" dynamodbav:"account_id"`
	AmountCents int64     `json:"amount_cents" dynamodbav:"amount_cents"`
	Note        string    `json:"note" dynamodbav:"note"`
	ExpiresAt   *time.Time `json:"expires_at,omitempty" dynamodbav:"expires_at,omitempty"`
	CreatedAt   time.Time `json:"created_at" dynamodbav:"created_at"`
	CreatedBy   string    `json:"created_by" dynamodbav:"created_by"`
}

// RedeemableCode represents a one-time or multi-use code that users can redeem.
type RedeemableCode struct {
	Code        string    `json:"code" dynamodbav:"code"`
	AmountCents int64     `json:"amount_cents" dynamodbav:"amount_cents"`
	MaxUses     int       `json:"max_uses" dynamodbav:"max_uses"`
	CurrentUses int       `json:"current_uses" dynamodbav:"current_uses"`
	ExpiresAt   *time.Time `json:"expires_at,omitempty" dynamodbav:"expires_at,omitempty"`
	CreatedAt   time.Time `json:"created_at" dynamodbav:"created_at"`
	CreatedBy   string    `json:"created_by" dynamodbav:"created_by"`
}

// InvoiceApplication represents the result of applying credits to an invoice.
type InvoiceApplication struct {
	InvoiceID       string `json:"invoice_id"`
	AccountID       string `json:"account_id"`
	TotalCents      int64  `json:"total_cents"`
	CreditAppliedCents int64 `json:"credit_applied_cents"`
	RemainingCents  int64  `json:"remaining_cents"`
	StripeChargeID  string `json:"stripe_charge_id,omitempty"`
}

// ============================================================================
// Repository Interface
// ============================================================================

// CreditRepository defines the data access interface for credit operations.
type CreditRepository interface {
	CreateCreditGrant(ctx context.Context, grant *CreditGrant) error
	GetCreditGrants(ctx context.Context, accountID string) ([]*CreditGrant, error)
	GetAvailableCredit(ctx context.Context, accountID string) (int64, error)
	ConsumeCredit(ctx context.Context, accountID string, amountCents int64) error

	CreateRedeemableCode(ctx context.Context, code *RedeemableCode) error
	GetRedeemableCode(ctx context.Context, code string) (*RedeemableCode, error)
	IncrementCodeUsage(ctx context.Context, code string) error
}

// ============================================================================
// DynamoDB Repository Implementation
// ============================================================================

type DynamoDBCreditRepository struct {
	client    *dynamodb.DynamoDB
	tableName string
	logger    *zap.Logger
}

func NewDynamoDBCreditRepository(client *dynamodb.DynamoDB, tableName string, logger *zap.Logger) *DynamoDBCreditRepository {
	return &DynamoDBCreditRepository{
		client:    client,
		tableName: tableName,
		logger:    logger,
	}
}

func (r *DynamoDBCreditRepository) CreateCreditGrant(ctx context.Context, grant *CreditGrant) error {
	const op = "DynamoDBCreditRepository.CreateCreditGrant"

	if grant.ID == "" {
		grant.ID = uuid.New().String()
	}
	if grant.CreatedAt.IsZero() {
		grant.CreatedAt = time.Now().UTC()
	}

	item, err := dynamodbattribute.MarshalMap(grant)
	if err != nil {
		r.logger.Error("failed to marshal credit grant", zap.String("operation", op), zap.Error(err))
		return fmt.Errorf("%s: marshal grant: %w", op, err)
	}

	input := &dynamodb.PutItemInput{
		TableName: aws.String(r.tableName),
		Item:      item,
	}

	if _, err := r.client.PutItemWithContext(ctx, input); err != nil {
		r.logger.Error("failed to create credit grant",
			zap.String("operation", op),
			zap.String("account_id", grant.AccountID),
			zap.Error(err),
		)
		return fmt.Errorf("%s: put item: %w", op, err)
	}

	r.logger.Info("credit grant created",
		zap.String("grant_id", grant.ID),
		zap.String("account_id", grant.AccountID),
		zap.Int64("amount_cents", grant.AmountCents),
	)

	return nil
}

func (r *DynamoDBCreditRepository) GetCreditGrants(ctx context.Context, accountID string) ([]*CreditGrant, error) {
	const op = "DynamoDBCreditRepository.GetCreditGrants"

	input := &dynamodb.QueryInput{
		TableName:              aws.String(r.tableName),
		KeyConditionExpression: aws.String("account_id = :account_id"),
		ExpressionAttributeValues: map[string]*dynamodb.AttributeValue{
			":account_id": {S: aws.String(accountID)},
		},
	}

	result, err := r.client.QueryWithContext(ctx, input)
	if err != nil {
		r.logger.Error("failed to query credit grants",
			zap.String("operation", op),
			zap.String("account_id", accountID),
			zap.Error(err),
		)
		return nil, fmt.Errorf("%s: query: %w", op, err)
	}

	var grants []*CreditGrant
	if err := dynamodbattribute.UnmarshalListOfMaps(result.Items, &grants); err != nil {
		r.logger.Error("failed to unmarshal credit grants",
			zap.String("operation", op),
			zap.Error(err),
		)
		return nil, fmt.Errorf("%s: unmarshal: %w", op, err)
	}

	return grants, nil
}

func (r *DynamoDBCreditRepository) GetAvailableCredit(ctx context.Context, accountID string) (int64, error) {
	const op = "DynamoDBCreditRepository.GetAvailableCredit"

	grants, err := r.GetCreditGrants(ctx, accountID)
	if err != nil {
		return 0, fmt.Errorf("%s: get grants: %w", op, err)
	}

	var totalAvailable int64
	now := time.Now().UTC()

	for _, grant := range grants {
		if grant.ExpiresAt != nil && now.After(*grant.ExpiresAt) {
			continue // Skip expired grants
		}
		totalAvailable += grant.AmountCents
	}

	return totalAvailable, nil
}

func (r *DynamoDBCreditRepository) ConsumeCredit(ctx context.Context, accountID string, amountCents int64) error {
	const op = "DynamoDBCreditRepository.ConsumeCredit"

	grants, err := r.GetCreditGrants(ctx, accountID)
	if err != nil {
		return fmt.Errorf("%s: get grants: %w", op, err)
	}

	remainingToConsume := amountCents
	now := time.Now().UTC()

	for _, grant := range grants {
		if remainingToConsume <= 0 {
			break
		}

		if grant.ExpiresAt != nil && now.After(*grant.ExpiresAt) {
			continue
		}

		consumeAmount := minInt64(grant.AmountCents, remainingToConsume)

		// Update grant in DynamoDB
		input := &dynamodb.UpdateItemInput{
			TableName: aws.String(r.tableName),
			Key: map[string]*dynamodb.AttributeValue{
				"id": {S: aws.String(grant.ID)},
			},
			UpdateExpression: aws.String("SET amount_cents = amount_cents - :consume"),
			ExpressionAttributeValues: map[string]*dynamodb.AttributeValue{
				":consume": {N: aws.String(fmt.Sprintf("%d", consumeAmount))},
			},
			ConditionExpression: aws.String("amount_cents >= :consume"),
		}

		if _, err := r.client.UpdateItemWithContext(ctx, input); err != nil {
			r.logger.Error("failed to consume credit",
				zap.String("operation", op),
				zap.String("grant_id", grant.ID),
				zap.Int64("consume_amount", consumeAmount),
				zap.Error(err),
			)
			return fmt.Errorf("%s: update grant %s: %w", op, grant.ID, err)
		}

		remainingToConsume -= consumeAmount
	}

	if remainingToConsume > 0 {
		r.logger.Warn("insufficient credit to fully consume",
			zap.String("operation", op),
			zap.String("account_id", accountID),
			zap.Int64("remaining", remainingToConsume),
		)
		return ErrInsufficientCredit
	}

	return nil
}

func (r *DynamoDBCreditRepository) CreateRedeemableCode(ctx context.Context, code *RedeemableCode) error {
	const op = "DynamoDBCreditRepository.CreateRedeemableCode"

	if code.Code == "" {
		generatedCode, err := generateSecureCode(16)
		if err != nil {
			r.logger.Error("failed to generate secure code", zap.String("operation", op), zap.Error(err))
			return fmt.Errorf("%s: generate code: %w", op, err)
		}
		code.Code = generatedCode
	}
	if code.CreatedAt.IsZero() {
		code.CreatedAt = time.Now().UTC()
	}

	item, err := dynamodbattribute.MarshalMap(code)
	if err != nil {
		r.logger.Error("failed to marshal redeemable code", zap.String("operation", op), zap.Error(err))
		return fmt.Errorf("%s: marshal code: %w", op, err)
	}

	input := &dynamodb.PutItemInput{
		TableName: aws.String(r.tableName),
		Item:      item,
		ConditionExpression: aws.String("attribute_not_exists(code)"),
	}

	if _, err := r.client.PutItemWithContext(ctx, input); err != nil {
		r.logger.Error("failed to create redeemable code",
			zap.String("operation", op),
			zap.String("code", code.Code),
			zap.Error(err),
		)
		return fmt.Errorf("%s: put item: %w", op, err)
	}

	r.logger.Info("redeemable code created",
		zap.String("code", code.Code),
		zap.Int64("amount_cents", code.AmountCents),
		zap.Int("max_uses", code.MaxUses),
	)

	return nil
}

func (r *DynamoDBCreditRepository) GetRedeemableCode(ctx context.Context, code string) (*RedeemableCode, error) {
	const op = "DynamoDBCreditRepository.GetRedeemableCode"

	input := &dynamodb.GetItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]*dynamodb.AttributeValue{
			"code": {S: aws.String(code)},
		},
	}

	result, err := r.client.GetItemWithContext(ctx, input)
	if err != nil {
		r.logger.Error("failed to get redeemable code",
			zap.String("operation", op),
			zap.String("code", code),
			zap.Error(err),
		)
		return nil, fmt.Errorf("%s: get item: %w", op, err)
	}

	if result.Item == nil {
		return nil, ErrCodeNotFound
	}

	var redeemableCode RedeemableCode
	if err := dynamodbattribute.UnmarshalMap(result.Item, &redeemableCode); err != nil {
		r.logger.Error("failed to unmarshal redeemable code",
			zap.String("operation", op),
			zap.Error(err),
		)
		return nil, fmt.Errorf("%s: unmarshal: %w", op, err)
	}

	return &redeemableCode, nil
}

func (r *DynamoDBCreditRepository) IncrementCodeUsage(ctx context.Context, code string) error {
	const op = "DynamoDBCreditRepository.IncrementCodeUsage"

	input := &dynamodb.UpdateItemInput{
		TableName: aws.String(r.tableName),
		Key: map[string]*dynamodb.AttributeValue{
			"code": {S: aws.String(code)},
		},
		UpdateExpression: aws.String("SET current_uses = current_uses + :inc"),
		ConditionExpression: aws.String("current_uses < max_uses"),
		ExpressionAttributeValues: map[string]*dynamodb.AttributeValue{
			":inc": {N: aws.String("1")},
		},
	}

	if _, err := r.client.UpdateItemWithContext(ctx, input); err != nil {
		r.logger.Error("failed to increment code usage",
			zap.String("operation", op),
			zap.String("code", code),
			zap.Error(err),
		)
		return fmt.Errorf("%s: update item: %w", op, err)
	}

	return nil
}

// ============================================================================
// Service Layer
// ============================================================================

// CreditService handles business logic for credit operations.
type CreditService struct {
	repo   CreditRepository
	logger *zap.Logger
}

func NewCreditService(repo CreditRepository, logger *zap.Logger) *CreditService {
	return &CreditService{
		repo:   repo,
		logger: logger,
	}
}

// ApplyCreditGrant applies a credit grant to a billing account.
func (s *CreditService) ApplyCreditGrant(ctx context.Context, accountID string, amountCents int64, note string, createdBy string, expiresAt *time.Time) (*CreditGrant, error) {
	const op = "CreditService.ApplyCreditGrant"

	if err := validateAccountID(accountID); err != nil {
		return nil, fmt.Errorf("%s: %w", op, err)
	}
	if err := validateAmountCents(amountCents); err != nil {
		return nil, fmt.Errorf("%s: %w", op, err)
	}
	if err := validateNote(note); err != nil {
		return nil, fmt.Errorf("%s: %w", op, err)
	}
	if err := validateCreatedBy(createdBy); err != nil {
		return nil, fmt.Errorf("%s: %w", op, err)
	}

	grant := &CreditGrant{
		AccountID:   accountID,
		AmountCents: amountCents,
		Note:        note,
		ExpiresAt:   expiresAt,
		CreatedBy:   createdBy,
	}

	if err := s.repo.CreateCreditGrant(ctx, grant); err != nil {
		return nil, fmt.Errorf("%s: create grant: %w", op, err)
	}

	return grant, nil
}

// RedeemCode redeems a code and applies the credit to the account.
func (s *CreditService) RedeemCode(ctx context.Context, code string, accountID string) (*CreditGrant, error) {
	const op = "CreditService.RedeemCode"

	if err := validateCode(code); err != nil {
		return nil, fmt.Errorf("%s: %w", op, err)
	}
	if err := validateAccountID(accountID); err != nil {
		return nil, fmt.Errorf("%s: %w", op, err)
	}

	redeemableCode, err := s.repo.GetRedeemableCode(ctx, code)
	if err != nil {
		return nil, fmt.Errorf("%s: get code: %w", op, err)
	}

	// Check if code is expired
	if redeemableCode.ExpiresAt != nil && time.Now().UTC().After(*redeemableCode.ExpiresAt) {
		return nil, ErrCodeExpired
	}

	// Check if code has remaining uses
	if redeemableCode.CurrentUses >= redeemableCode.MaxUses {
		return nil, ErrCodeExhausted
	}

	// Increment usage count
	if err := s.repo.IncrementCodeUsage(ctx, code); err != nil {
		return nil, fmt.Errorf("%s: increment usage: %w", op, err)
	}

	// Create credit grant
	grant := &CreditGrant{
		AccountID:   accountID,
		AmountCents: redeemableCode.AmountCents,
		Note:        fmt.Sprintf("Redeemed code: %s", code),
		CreatedBy:   "system",
	}

	if err := s.repo.CreateCreditGrant(ctx, grant); err != nil {
		// Attempt to rollback code usage increment
		if rollbackErr := s.repo.IncrementCodeUsage(ctx, code); rollbackErr != nil {
			s.logger.Error("failed to rollback code usage increment",
				zap.String("operation", op),
				zap.String("code", code),
				zap.Error(rollbackErr),
			)
		}
		return nil, fmt.Errorf("%s: create grant: %w", op, err)
	}

	return grant, nil
}

// ApplyCreditsToInvoice applies available credits to an invoice before charging.
func (s *CreditService) ApplyCreditsToInvoice(ctx context.Context, invoiceID string, accountID string) (*InvoiceApplication, error) {
	const
# Resource Management Platform

## Overview

This document outlines the vision and proposed enhancements for the Milo
platform resource management system. This system provides a business-oriented
resource management foundation that enables service providers to define,
manage, and govern any type of business resource—from services and subscriptions
to contracts and assets—throughout their entire lifecycle.

The platform serves as the system of record for all business resources,
providing hierarchical multi-tenancy, extensible resource types, declarative
APIs, and integration points for billing, IAM, workflow automation, and other
critical business systems.

> [!IMPORTANT]
>
> **Status**: This is a collection of enhancement proposals. The core hierarchy
> features are in provisional status while advanced capabilities described are
> in planning stages and not yet implemented.

## Current Status

### Implemented Features

- **Core Resource Hierarchy**: Organizations and Projects structure with
  hierarchical IAM
- **Quota Management**: Hierarchical allocation, enforcement, and monitoring
- **Garbage Collection**: Orphaned resource cleanup and cascading deletion
- **Resource Discovery & Registration**: Service catalog integration, dynamic
  resource type registration, capability discovery, and schema evolution
- **Admission Control Policies**: Declarative validation and mutation policies,
  business rule enforcement, auto-tagging, and naming conventions
- **Watch & Events**: Real-time change notifications and event recording for resources
- **Optimistic Concurrency Control**: Resource version and generation tracking
- **Pre-Deletion Hooks**: Framework for custom cleanup tasks before resource removal (e.g., archiving, notifications)
- **Metadata Queries**: Field and label selectors for resource filtering

## Planned Capabilities

- **Advanced Search**: Full-text search, faceted filtering, aggregation queries,
  and RBAC-protected indexed search across all resource types
- **Lifecycle Policies**: Automated cleanup and expiration with configurable
  retention rules
- **Resource protection**: Protect resources by preventing inappropriate actions
  (e.g. deletions)
- **Bundles & Templates**: Composable resource packages with templating,
  overlays, parameterization, and atomic deployment (similar to [Helm]/[Kustomize])
- **Knowledge Graph**: Resource relationship APIs, bidirectional traversal, and
  typed relationships
- **Resource Archiving**: Long-term retention for compliance and recovery
- **Revision History**: Configuration change tracking with rollback capabilities
- **Complex Resource Hierarchy**: Folder hierarchy support for intermediate
  organizational structure

[Helm]: https://helm.sh
[Kustomize]: https://kustomize.io

## Planned Integration Points

The resource management system will integrate with several platform components:

- **IAM System**: Hierarchical access control and permission inheritance
- **Billing Platform**: Project-level cost allocation and resource-based billing
- **Service Catalog**: Resource type definitions and service integration
- **Telemetry Platform**: Resource health metrics and usage analytics
- **Communication Platform**: Alert delivery for resource events and policy
  violations
- **Audit System**: Comprehensive tracking of resource lifecycle events

## Target Use Cases

The resource management platform aims to support diverse organizational needs:

### Individual Developers

- Simple Organization with single Project
- Minimal hierarchy complexity
- Easy resource discovery and management

### Small Organizations

- Multiple Projects for different applications
- Team-based access control
- Flexible project reorganization

### Enterprises

- Complex folder hierarchies mirroring business structure
- Delegated administration per business unit
- Sophisticated compliance and governance requirements

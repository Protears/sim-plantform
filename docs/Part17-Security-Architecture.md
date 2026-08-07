# IW.SIM Part17 - Security Architecture

## Overview

IW.SIM Security Architecture provides identity, authentication, authorization and runtime protection for industrial simulation environments.

## Security Layers

- Application Security
- API Security
- Runtime Isolation
- Communication Security
- Data Security

## Identity

Support:

- User identity
- Project identity
- Device identity
- Service identity

## Authorization Model

RBAC + Resource Permission:

- Project permission
- Scene permission
- Device operation permission
- Test execution permission

## Communication Protection

- HTTPS
- JWT Token
- TLS
- Internal network isolation

## Industrial Safety

Simulation commands require:

- Permission validation
- State validation
- Safety interlock checking

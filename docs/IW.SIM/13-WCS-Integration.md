# WCS Integration Design

## Overview
IW.SIM provides a virtual commissioning environment for WCS scheduling systems.

## Integration Model

WCS communicates with IW.SIM through industrial protocols and simulation APIs.

## Supported Functions

- Task creation
- Device dispatch
- Status feedback
- Cargo tracking
- Exception handling

## Runtime Flow

WCS Command -> Simulation Gateway -> Device Runtime -> World State Update -> Feedback

## Design Goals

- Protocol compatibility with real equipment
- Deterministic test execution
- Support automated regression testing

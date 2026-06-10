# test

DCP Resolution Engine — Current Handoff
Goal

Build a Spring Boot microservice that receives slots from a Python banking assistant and resolves them against reference values using exact match and fuzzy match.

The service is a generic resolution engine, not a workflow engine.

Example input:

{
  "workflow": "new_payment",
  "slots": [
    { "slotType": "ACCOUNT", "value": "abc.savings" },
    { "slotType": "BENEFICIARY", "value": "jonh" },
    { "slotType": "CURRENCY", "value": "INR" },
    { "slotType": "PAYMENT_METHOD", "value": "RTGS" }
  ]
}

The Python assistant:

detects intent
selects workflow
extracts slots
sends them to this engine

This Java service:

resolves slot values
returns exact/fuzzy results
returns suggestions
remains workflow-agnostic
Final stack

Use:

Java 21
Spring Boot 4.x
Gradle Groovy DSL

Spring Initializr dependencies:

Spring Web
Validation
Spring Security
Spring Boot Actuator
Lombok

Do not add Prometheus, Swagger, Resilience4j, DB, Redis, Kafka, Feign, Eureka, Config Server yet.

Final package structure

Use normal layered Spring architecture:

com.bank.dcp.resolutionengine

├── controller
├── service
├── datasource
├── model
├── exception
├── advice
├── config
├── security
└── util

Responsibilities:

controller → HTTP only
service → business logic only
datasource → reference data access only
model → DTOs/enums
exception → custom exceptions
advice → global exception handling
config → Spring config
security → filters/security-related code
Naming

Project coordinates:

Group: com.bank.dcp
Artifact: dcp-resolution-engine
Package: com.bank.dcp.resolutionengine

Main class:

package com.bank.dcp.resolutionengine;

@SpringBootApplication
public class DcpResolutionEngineApplication {}
Current design decision

Use a generic slot list and a workflow field.

Do not hardcode workflow-specific fields in the request body.

The request body should stay workflow-agnostic and extensible.

Current request contract
Headers

Keep transport/trace metadata in headers:

X-Correlation-Id
X-Request-Id

Later you can add:

X-User-Id
X-Session-Id
X-Tenant-Id
X-Source-Service

For now, the body carries the user identifier because the reference lookup is user-scoped.

Request body

Use:

public record ResolutionRequest(
        WorkflowType workflow,
        String userId,
        List<SlotRequest> slots
) {}
Slot request
public record SlotRequest(
        SlotType slotType,
        String value
) {}
Workflow type
public enum WorkflowType {
    NEW_PAYMENT,
    CHECK_BALANCE,
    REPEAT_PAYMENT,
    PAY_USING_TEMPLATE
}
Slot type
public enum SlotType {
    ACCOUNT,
    BENEFICIARY,
    PAYMENT_METHOD,
    CURRENCY,
    TEMPLATE_NAME,
    AMOUNT
}
Why this request shape

This design is meant to support future workflows without changing the API.

Examples:

NEW_PAYMENT may need ACCOUNT, BENEFICIARY, CURRENCY, PAYMENT_METHOD
CHECK_BALANCE may need only ACCOUNT
REPEAT_PAYMENT may need ACCOUNT, BENEFICIARY, AMOUNT, CURRENCY

The engine should not know these rules as if/else logic. The Python side decides which slots to send.

Current response contract
Single-slot result
public record ResolutionResult(
        SlotType slotType,
        String inputValue,
        boolean verified,
        boolean exactMatch,
        String canonicalValue,
        List<String> suggestions
) {}
Full response
public record ResolutionResponse(
        boolean allVerified,
        List<ResolutionResult> results
) {}

This structure keeps the response ordered and easy to consume.

Example response
{
  "allVerified": false,
  "results": [
    {
      "slotType": "ACCOUNT",
      "inputValue": "abc.savings",
      "verified": true,
      "exactMatch": true,
      "canonicalValue": "abc.savings",
      "suggestions": []
    },
    {
      "slotType": "BENEFICIARY",
      "inputValue": "jonh",
      "verified": false,
      "exactMatch": false,
      "canonicalValue": null,
      "suggestions": ["john"]
    }
  ]
}
Error handling design

Use both packages:

exception

Contains exception classes only:

ResolutionException
DataSourceException
ResourceNotFoundException
ValidationException
advice

Contains:

GlobalExceptionHandler

This handler should translate exceptions into consistent API error responses.

Also handle validation failures such as:

MethodArgumentNotValidException
Coding standards

Follow these rules:

constructor injection only
use @RequiredArgsConstructor
no field injection
use record for DTOs
no System.out.println
use LoggerFactory
validate request inputs at the API boundary with @Valid
keep controller thin
keep service focused on business logic
keep datasource focused only on retrieval
Architecture rule

Use a normal layered architecture, not full hexagonal/clean architecture for now.

Avoid:

full ports/adapters setup
CQRS
event sourcing
Kafka
service discovery
DB-first design

Reason: this is still a POC and the concrete data source is not fixed yet.

Build order so far
Create project
Create package structure
Create SlotType
Create WorkflowType
Create SlotRequest
Create ResolutionRequest
Create ResolutionResult
Create ResolutionResponse
Create ReferenceDataSource
Add in-memory or file-backed implementation
Add exact match service
Add fuzzy match service
Add resolution service
Add controller
Add global exception handling
Add security and tracing
Add Actuator enhancements later
Next class to create

The next sensible class after the current DTOs is:

ReferenceDataSource


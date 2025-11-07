# API Evolve Policies

## Overview

API Evolve Policies is a powerful feature in AtomicLinkRPC (ALR) that helps manage API evolution by verifying runtime access patterns on struct fields, method parameters, and return values. Unlike traditional versioning approaches that require global version numbers or complex migration logic, ALR's evolve policies work at the individual value level, allowing gradual API evolution without breaking changes.

Since ALR performs schema negotiation during the handshake, each endpoint knows exactly what fields, types, and methods the remote endpoint supports. Evolve policies leverage this knowledge to:

- **Enforce correct usage** of remote capability checks
- **Catch unintended API mismatches** during development
- **Provide detailed diagnostics** when these values are incorrectly accessed
- **Enable gradual API evolution** without breaking existing clients or servers
- **Validate enum values** to ensure both endpoints understand the data

## Core Concepts

### The `alr::Evolve<T>` Wrapper

The `Evolve<T, WritePolicy, ReadPolicy>` template wraps a value of type `T` and attaches write and read policies to it:

```cpp
template<class T, EvolvePolicy WritePolicy = EvolvePolicy::Required,
                  EvolvePolicy ReadPolicy = EvolvePolicy::Optional>
class Evolve { /* ... */ };
```

**Key characteristics:**

- Behaves like `T` for most operations (implicit conversions)
- Tracks whether the value has been written or read
- Enforces policies at runtime based on remote capabilities
- Zero overhead when policies compile away (using type aliases like `using MyPolicy<T> = T`)
- Can be applied to struct fields, method parameters, and return values

### Write vs Read Policies

- **Write Policy**: Applied when sending data to the remote endpoint. Enforces write requirements before transmission.
- **Read Policy**: Applied when receiving data from the remote endpoint. Enforces read requirements after reception.

**Note**: Method parameters and return values are always marked as "written" since they are always supplied, however all other policy enforcement (read verification, enum validation, etc.) applies to them.

### Remote Capability Awareness

Each policy mode can be **capability-aware**, meaning it adjusts its behavior based on whether the remote endpoint knows about the value:

- **Remote has value**: The remote endpoint's schema includes this value
- **Remote lacks value**: The remote endpoint's schema doesn't include this value

## Policy Modes

### Base Policy Modes

The following table describes the base policy modes (without enum-specific validation):

| Policy Mode| Write Behavior | Read Behavior |
|-------------|----------------|---------------|
| `Optional` | Write is optional. If unset, sends local default. | Read is optional. If remote lacks value, uses local default. |
| `RequiredIfKnown` | **Must write if remote knows** the value. Otherwise optional. | **Must read if remote knows** the value. Otherwise optional. |
| <nobr>`ProhibitedIfUnknown`</nobr> | Optional if remote knows. **Prohibited if remote doesn't know**. | Optional if remote knows. **Prohibited if remote doesn't know**. |
| `FollowsKnown` | Must write if remote knows, prohibited otherwise (detects missed caps check). | Must read if remote knows, prohibited otherwise (detects missed caps check). |
| `Required` | **Always require write**, even if unknown to remote. | **Always require read**, even if unknown to remote. |

### Enum Policy Modifiers

For enum types, additional validation can be applied:

| Enum Modifier | Validation |
|---------------|------------|
| `_EnumValue` | Remote must know the **exact enum value** (e.g., `MyEnum::Foo = 5`) |
| `_EnumStrict` | Remote must know the **exact value AND matching name** (detects refactoring) |
| `_EnumFlags` | Remote must know **all set flag bits** (for bitwise-OR enums) |
| `_EnumFlagsStrict` | Remote must know **all set flag bits with matching names** |

**Examples:**
```cpp
// Base policy + enum validation
EvolvePolicy::RequiredIfKnown_EnumValue
EvolvePolicy::Optional_EnumFlags
EvolvePolicy::Required_EnumStrict
```

### Common Policy Combinations

ALR provides convenient type aliases for common patterns:

```cpp
// Always require write, read is optional. For an enum, the remote must know the
// exact enum value. Use for new required fields during migration.
template<class T>
using EvolveWriteRequired = Evolve<T, EvolvePolicy::Required_EnumValue,
                                      EvolvePolicy::Optional_EnumValue>;

// Always require write and read. For an enum, the remote must know the exact
// enum value and name. Use when both endpoints must be synchronized.
template<class T>
using EvolveStrict = Evolve<T, EvolvePolicy::Required_EnumStrict,
                               EvolvePolicy::Required_EnumStrict>;

// Write if the remote knows, optional otherwise. Read is optional. For an enum,
// the remote must know the exact enum value. Use for gradual field rollout.
template<class T>
using EvolveOptional = Evolve<T, EvolvePolicy::RequiredIfKnown_EnumValue,
                                 EvolvePolicy::Optional_EnumValue>;

// Write and read are both required if remote knows, optional otherwise. For an
// enum, the remote must know the exact enum value. Use when field presence
// should match capability.
template<class T>
using EvolveIfKnown = Evolve<T, EvolvePolicy::RequiredIfKnown_EnumValue,
                                EvolvePolicy::RequiredIfKnown_EnumValue>;

// Write follows remote knowledge (must write if known, must be unset if unknown).
// Read is optional. For an enum, the remote must know the exact enum value.
// Use when sending new fields that require capability checks.
template<class T>
using EvolveFollowsCapability = Evolve<T, EvolvePolicy::FollowsKnown_EnumValue,
                                          EvolvePolicy::Optional_EnumValue>;

// Write is optional, but prohibited if remote doesn't know. Read is optional.
// For an enum, the remote must know the exact enum value. Use for optional
// fields that should never be sent to older endpoints.
template<class T>
using
EvolveProhibitIfUnknown = Evolve<T, EvolvePolicy::ProhibitedIfUnknown_EnumValue,
                                    EvolvePolicy::Optional_EnumValue>;

// Both write and read are optional. Use on deprecated fields as a convenient
// way to indicate they should not be used with any new code and will be removed
// soon.
template<class T>
using EvolveDeprecated = T;
```

## Usage Patterns

### Basic Field Declaration

```cpp
struct WeatherInfo
{
    // V1 field - deprecated, optional to read/write
    EvolveDeprecated<std::string> conditions;

    // V2 field - stable, must write and read if remote knows
    EvolveIfKnown<float> temperature;

    // V3 field - new, must write if remote knows, read is optional
    EvolveOptional<int> hour;
};
```

### Method Parameters and Return Values

Evolve policies can also be applied directly to method parameters and return values:

```cpp
class MyService : public alr::EndpointClass
{
public:
    // Policy on parameter - must be read if remote knows it
    void processData(alr::EvolveStrict<int> importantValue);
    
    // Policy on return value - validates remote knows enum value with matching name
    alr::EvolveStrict<ServiceStatus> getStatus();
};
```

**Key notes**:

- Parameters and return values that are known by the remote are always marked as "written" (since they must be supplied)
- Read policies still apply (e.g., the caller must read a return value if required)
- Enum validation policies work the same as for fields
- Useful for enforcing correct handling of critical parameters/returns across versions

### Using Type Aliases for Version Management

For better maintainability, define version-based type aliases:

```cpp
// Version 1: Deprecated fields
template<class T>
using V1_Deprecated = T;

// Version 2: Stable fields
template<class T>
using V2_Stable = alr::Evolve<T, EvolvePolicy::RequiredIfKnown,
                                 EvolvePolicy::RequiredIfKnown>;

// Version 3: New fields being introduced
template<class T>
using V3_New = alr::Evolve<T, EvolvePolicy::RequiredIfKnown,
                              EvolvePolicy::Optional>;

// Version 3: New enum fields with flag validation
template<class T>
using V3_NewEnumFlags = alr::Evolve<T, EvolvePolicy::RequiredIfKnown_EnumFlags,
                                       EvolvePolicy::Optional_EnumFlags>;

// Now use them:
struct WeatherInfo
{
    V1_Deprecated<std::string> conditions;
    V2_Stable<float> temperature;
    V3_New<int> hour;
    V3_NewEnumFlags<WeatherWarnings> warnings;
};
```

**Evolution over time:**
```cpp
// When V4 arrives:
// 1. V3_New → V3_Stable (fields are now established)
// 2. V2_Stable → V2_Deprecated (for old fields being phased out)
// 3. Define V4_New for new fields
// 4. Eventually remove V1_Deprecated fields entirely once V1 is no longer supported
```

### Accessing Fields

#### Implicit Read (Marks as Read)
```cpp
// Implicit conversion to const T&
float temp = weatherInfo.temperature;  // Marks as read

// Passing to function expecting T or const T&
void processTemp(const float& t);
// ...
processTemp(weatherInfo.temperature);  // Marks as read
```

#### Explicit Write (Marks as Written)
```cpp
// Explicit cast to T& for modification
float& tempRef = static_cast<float&>(weatherInfo.temperature);
tempRef = 72.5f;  // Marks as written

// Direct assignment
weatherInfo.temperature = 72.5f;       // Marks as written
```

#### Nested Structs
```cpp
struct WeatherConditions
{
    V2_Stable<float> windDirection;
    V2_Stable<float> windSpeed;
};

struct WeatherInfo
{
    WeatherConditions conditions;  // Prefer no policy on the struct itself
};

// Read-only access (no churn when adding/removing policies)
float dir = weatherInfo.conditions.windDirection;  // Marks windDirection as read

// Write access (no churn when adding/removing policies)
weatherInfo.conditions.windSpeed = 15.0f;  // Marks windSpeed as written
```

#### Default Values

Both method parameters and struct fields support default values, with or without policies:

```cpp
struct WeatherConditions
{
    V2_Stable<int> uvIndex = 5;
    V2_Stable<std::string> warnings = "No warnings";
};

class CityGuideService : public alr::EndpointClass
{
public:
    WeatherForecast getWeatherForecast(const Location& location,
                                       V2_Stable<int> hours = 6);
};
```

**How defaults work with evolve policies:**

- **Receiving unknown values**: When the remote endpoint doesn't know about a field or parameter, the local default value is used automatically
- **Sending unset values**: When you don't explicitly set a field or parameter before sending, the local default value is transmitted
- **Policy enforcement**: Policies apply regardless of defaults, e.g. if a field requires writing when remote knows it, you must explicitly set it even if it has a default value

This behavior ensures backward compatibility while still allowing policy enforcement when both endpoints support a field.

#### Using `evolve::read()` and `evolve::write()` Helpers

For uniform access patterns that work with both `T` and `Evolve<T>`:

```cpp
#include <alr/evolve.h>

// Read access - works for both plain T and Evolve<T>
const auto& temp = evolve::read(weatherInfo.temperature);

// Write access - works for both plain T and Evolve<T>
auto& tempRef = evolve::write(weatherInfo.temperature);
tempRef = 75.0f;

// Nested struct fields - works for both plain T and Evolve<T>
evolve::write(weatherInfo.conditions).windSpeed = 20.0f;
```

**Benefits:**

- Zero churn when adding or removing policies from fields
- Explicit intent (read vs write)
- Works uniformly across all field types

### Using Remote Capability Checks

ALR generates `remoteCaps` namespace functions during code generation. Use these to conditionally access values:

```cpp
// Check if remote has a specific field
if (remoteCaps::hasStructField::WeatherInfo::hour()) {
    weatherInfo.hour = 14;  // Safe to set
}

// Check if remote has an enum value
if (remoteCaps::hasEnum::WeatherWarnings(EnumCaps::Value, WeatherWarnings::HighWind)) {
    weatherInfo.warnings = WeatherWarnings::HighWind;
}

// Check if remote has all flag bits
WeatherWarnings flags = WeatherWarnings::Storm | WeatherWarnings::HighWind;
if (remoteCaps::hasEnum::WeatherWarnings(EnumCaps::Flags, flags)) {
    weatherInfo.warnings = flags;  // Remote knows all flags
} else {
    weatherInfo.warnings = WeatherWarnings::Storm;  // Use safe subset
}
```

**Enum remote capability check types:**
```cpp
enum class EnumCaps {
    Type,          // Check if remote has the enum type
    Value,         // Check if remote has a specific enum value
    ValueAndName,  // Check if remote has value and names match
    Flags,         // Check if remote has all set flag bits
    FlagsAndNames, // Check if remote has all set flag bits and names match
};
```

### Disabling Policy Checks on Error

If an RPC encounters an error and exits early, disable evolve checks to avoid cascading violations:

```cpp
AsyncVoid MyService::processData(const DataStruct& data)
{
    try {
        // Process data...
        if (someErrorCondition) {
            alr::CallCtx::disableEvolvePolicyChecks();
            return;  // Early exit, don't enforce policies
        }
        
        // Normal processing...
        float value = data.importantField;  // Will check policy
    }
    catch (...) {
        alr::CallCtx::disableEvolvePolicyChecks();
        throw;
    }
}
```

## Policy Violation Handling

### Violation Callbacks

When a policy is violated, ALR invokes the following callbacks:

```cpp
namespace alr
{
    class Callbacks
    {
    public:
        virtual void evolvePolicyViolated(
            EndpointWeakRef ep,
            const EvolvePolicyViolation& violation);

        virtual void log(EndpointWeakRef ep,
                         LogLevel level,
                         const std::string& msg);
    };
}
```

**Default behavior**: `alr::Callbacks::evolvePolicyViolated` logs a detailed error message to the current active logger (default is console).

**Custom behavior**: Override the callbacks to:

- Log to a custom logging system
- Capture telemetry/metrics
- Put the endpoint into a faulted state
- Ignore certain violations (not recommended)
- Throw exceptions

### Violation Details

The `alr::EvolvePolicyViolation` struct contains comprehensive information:

```cpp
struct EvolvePolicyViolation
{
    std::string detail;         // Detailed description with fix suggestions
    std::string methodName;     // Method being invoked
    std::string structName;     // Structure containing the field (if applicable)
    std::string valueName;      // Parameter or field name
    std::string valueType;      // The type of the parameter, field or return value
    std::string capsCheck;      // Suggested capability check (if applicable)
    
    std::string localEnumName;  // Local enum value name (if applicable)
    std::string remoteEnumName; // Remote enum value name (if applicable)
    int64_t enumValue;          // Enum value (if applicable)
    
    EvolvePolicy writePolicy;   // The write policy in effect
    EvolvePolicy readPolicy;    // The read policy in effect
    
    bool remoteHas;            // Remote has this field
    bool remoteHasEvolve;      // Remote field also has a policy
    bool isReceived;           // Receiving (true) or sending (false)
    bool isField;              // True if this is a field (not a param/return)
    bool isReturn;             // True if this is part of the return value (direct or nested)
    bool isReadRequired;       // Policy requires read
    bool isWriteRequired;      // Policy requires write
    bool isSet;                // Value was set (no longer the user-supplied default)
    bool isWritten;            // Value was written
    bool isRead;               // Value was read
};
```

## Example Violations

### Write Policy Violation

**Scenario**: Field requires writing when remote knows it, but wasn't written.

```
*********************************
* Evolve Policy Write Violation *
*********************************
Write Policy: alr::EvolvePolicy::RequiredIfKnown
Method:       cityguide::CityGuideService::getWeatherForecast( ... )
Struct:       cityguide::WeatherConditions (as return value)
Field:        float windSpeed
Remote has:   true
Was written:  false

Problem:
  Field 'cityguide::WeatherConditions::windSpeed' requires writing when remote knows it.

Fix:
  Use the following caps check:

  if (remoteCaps::hasStructField::cityguide::WeatherConditions::windSpeed()) {
     obj.windSpeed = value;
  }

Alternative fix:
  Relax (or remove) the policy to send the local default if the write is optional:
  alr::Evolve<T, EvolvePolicy::Optional, ...>
```

### Read Policy Violation

**Scenario**: Field requires reading when remote knows it, but wasn't read.

```
********************************
* Evolve Policy Read Violation *
********************************
Read Policy:  alr::EvolvePolicy::RequiredIfKnown
Method:       cityguide::CityGuideService::getWeatherInfo( ... )
Struct:       cityguide::WeatherConditions (as return value)
Field:        float windDirection
Remote has:   true
Was read:     false

Problem:
  Field 'cityguide::WeatherConditions::windDirection' requires reading when remote knows it.

Fix:
  Use the following caps check:

  if (remoteCaps::hasStructField::cityguide::WeatherConditions::windDirection()) {
     float value = obj.windDirection;
  }

Alternative fix:
  Relax (or remove) the policy if the read is optional:
  alr::Evolve<T, ..., EvolvePolicy::Optional>
```

### Enum Flags Violation

**Scenario**: Enum flags sent to remote, but remote doesn't know all flag bits.

```
*********************************
* Evolve Policy Write Violation *
*********************************
Write Policy: alr::EvolvePolicy::RequiredIfKnown_EnumFlags
Method:       cityguide::CityGuideService::getWeatherForecast( ... )
Struct:       cityguide::WeatherInfo (as return value)
Field:        cityguide::WeatherWarnings warnings
Remote has:   true
Was written:  true

Problem:
  An attempt was made to send enum 'cityguide::WeatherWarnings' with bit flags
  that are unknown to the remote endpoint:
  Set bits in enum value: 0b11100
  Local matching flags:   0b11100 (Storm=4 | Flood=8 | HighWind=16)
  Remote matching flags:  0b01100 (Storm=4 | Flood=8)

Fix:
  Use the following caps check to verify the remote endpoint supports all flags:

  if (remoteCaps::hasEnum::cityguide::WeatherWarnings(alr::EnumCaps::Flags, flags)) {
     obj.warnings = flags;
  } else {
     obj.warnings = altFlags; // Use a safe set of supported flags
  }

Alternative fix:
  Relax (or remove) the policy if is safe for the remote to ignore unknown enum flags:
  alr::Evolve<T, EvolvePolicy::Optional, ...>
```

## Best Practices

### When to Use Evolve Policies

✅ **Use policies for:**

- Fields that are likely to change over time
- Critical fields where incorrect usage could cause bugs
- Enum fields where version mismatches are risky
- Fields in widely-used, shared structs
- Return values that must be read by the caller
- Important method parameters that require validation

❌ **Don't use policies for:**

- Every field, parameter, or return value (adds verbosity and overhead)
- Internal implementation details
- Temporary/debug fields
- Fields in stable, unchanging structs

### Granularity Guidelines

**Prefer leaf fields over nested struct fields:**

```cpp
// GOOD: Policies on leaf fields
struct WeatherConditions
{
    V2_Stable<float> windDirection;
    V2_Stable<float> windSpeed;
};

struct WeatherInfo
{
    WeatherConditions conditions;  // No policy here
};

// LESS GOOD: Policy on nested struct field
struct WeatherInfo
{
    V2_Stable<WeatherConditions> conditions;  // Less granular control, more verbose access
};
```

**Consider policies on parameters/returns for critical APIs:**

```cpp
// GOOD: Policy on critical return value
alr::EvolveStrict<AuthToken> authenticate(const Credentials& creds);

// Also good: Policy on important parameter
void setPermissions(alr::EvolveStrict<PermissionFlags> perms);
```

**Benefits of targeted policies:**

- More granular control
- Fewer false positives
- Clearer violation messages
- Easier to evolve individual values
- Simple access patterns

### Performance Considerations

Evolve policies add minimal overhead (~33ns per non-violated policy check on hardware listed in benchmarks section), but you can optimize further:

**1. Use policies selectively**
```cpp
struct LargeStruct
{
    // Critical field with policy
    V2_Stable<int> criticalId;
    
    // Stable fields without policies
    int stableField1;
    int stableField2;
    std::string stableField3;
};
```

**2. Conditional compilation**
```cpp
#ifdef ENABLE_EVOLVE_POLICIES
template <typename T>
using MyPolicy = alr::Evolve<T,
    alr::EvolvePolicy::RequiredIfKnown,
    alr::EvolvePolicy::RequiredIfKnown>;
#else
template <typename T>
using MyPolicy = T;  // Zero overhead
#endif

struct MyStruct
{
    MyPolicy<int> field;  // Policy only when enabled
};
```

**3. Relax policies in production**
```cpp
// Development/Testing
template<class T>
using DevPolicy = alr::Evolve<T,
    EvolvePolicy::Required_EnumStrict,
    EvolvePolicy::Required_EnumStrict>;

// Production (more lenient)
template<class T>
using ProdPolicy = alr::Evolve<T,
    EvolvePolicy::Optional,
    EvolvePolicy::Optional>;

#ifdef DEBUG_BUILD
template<class T> using MyPolicy = DevPolicy<T>;
#else
template<class T> using MyPolicy = ProdPolicy<T>;
#endif
```

### Copy Tracking

For both parameters and for the return value, ALR tracks copies of values with required read policies to avoid false violations:

```cpp
AsyncRef<WeatherInfo> getWeather()
{
    WeatherInfo info;
    info.temperature = 72.5f;
    
    // Return by value - copy is tracked
    return info;
    
    // If client reads temperature from ANY copy,
    // the read requirement is satisfied
}

// Client side
void processWeather()
{
    auto info = getWeather();        // Copy 1
    WeatherInfo localCopy = info;    // Copy 2
    
    // Reading from ANY copy satisfies the requirement
    float temp = localCopy.temperature;  // ✓ Read satisfied
}
```

### Version Migration Strategy

**Step 1: Introduce new field (V2)**
```cpp
template<class T>
using V2_New = alr::Evolve<T,
    EvolvePolicy::RequiredIfKnown,
    EvolvePolicy::Optional>;

struct MyStruct
{
    V2_New<int> newField;
};
```

**Step 2: After adoption, make it stable (V3)**
```cpp
template<class T>
using V2_New = alr::Evolve<T,
    EvolvePolicy::RequiredIfKnown,
    EvolvePolicy::RequiredIfKnown>;  // Now require read too
```

**Step 3: Deprecate old fields (V4)**
```cpp
template<class T>
using V1_Deprecated = T;

struct MyStruct
{
    V1_Deprecated<int> oldField;  // Mark for removal
    int newField;  // No policy needed, now stable
};
```

**Step 4: Remove deprecated fields (V5)**
```cpp
struct MyStruct
{
    // oldField removed
    int newField;
};
```

## Advanced Topics

### Runtime Policy Modification

Policies can be changed at runtime if needed:

```cpp
void handleSpecialCase(WeatherInfo& info)
{
    // Temporarily relax policy
    evolve::setWritePolicy(info.temperature, EvolvePolicy::Optional);
    
    // ... special processing ...
    
    // Restore policy
    evolve::setWritePolicy(info.temperature, EvolvePolicy::RequiredIfKnown);
}
```

### Querying Policy State

```cpp
// Check if field has a policy
if (evolve::hasPolicy(info.temperature)) {
    auto writePolicy = evolve::getWritePolicy(info.temperature);
    auto readPolicy = evolve::getReadPolicy(info.temperature);
}
```

### Nested Field Policy Propagation

When a nested field is read or written, the parent field is also marked:

```cpp
struct Inner
{
    V2_Stable<int> value;
};

struct Outer
{
    V2_Stable<Inner> inner;
};

Outer obj;
auto& innerRef = evolve::write(obj.inner);
innerRef.value = 42;  // Marks both obj.inner AND inner.value as written
```

This reduces false negatives when nested structs are accessed through intermediate references.

## Summary

API Evolve Policies provide fine-grained control over API evolution in ALR:

- **Flexible**: Policies range from optional to strictly required
- **Granular**: Apply policies to individual fields, not entire structs
- **Informative**: Detailed violation messages with actionable fixes
- **Efficient**: Minimal overhead, can be compiled away if needed
- **Safe**: Catches version mismatches at runtime before they cause bugs
- **Evolvable**: Easy to add, modify, or remove policies as APIs mature

By leveraging remote capability checks and evolve policies together, you can build robust, evolvable RPC APIs that gracefully handle version mismatches across distributed systems.

## Protecting Against Breaking Changes During Development

A powerful use of evolve policies is **protecting against accidental breaking changes** when adding new features that involve extensive modifications:

**Scenario**: You're developing a new feature that adds several enum values and flags. You want to ensure backward compatibility with released versions.

**Strategy**:
```cpp
// During development only - add strict policies to enum fields
#ifdef DEVELOPMENT_BUILD
template<class T>
using DevEnumPolicy = alr::Evolve<T,
    EvolvePolicy::Required_EnumFlagsStrict,  // Strict write validation
    EvolvePolicy::Required_EnumFlagsStrict>; // Strict read validation
#else
template<class T>
using DevEnumPolicy = T;  // No overhead in production
#endif

struct MyStruct
{
    // Apply during development to detect breaking changes
    DevEnumPolicy<MyFlags> flags;
    DevEnumPolicy<MyStatus> status;
};
```

**Key benefits**:

1. **Catches refactoring errors** - Detects if you accidentally:

    - Reorder enum values (shifting numeric values)
    - Rename enum values (i.e. their meaning) without updating both endpoints
    - Add new flags that conflict with existing bits
   
2. **One-sided enforcement** - You only need to add policies to the **new version** being developed:

    - Write policies ensure you don't send incompatible data to old versions
    - Read policies ensure you handle data from old versions correctly
    - No changes needed to released versions

3. **Testing matrix integration** - Run your test suite against:

    - Latest development build (both sides with policies)
    - Development build vs. released versions (one side with policies)
    - Any policy violations immediately reveal breaking changes

4. **Zero production overhead** - Policies compile away in release builds

**Example workflow**:
```cpp
// Step 1: Add new enum flags in development
enum class FeatureFlags {
    Basic = 1 << 0,
    Advanced = 1 << 1,
    // New in v2.0 development
    Premium = 1 << 2,    // ✓ Safe - new bit
    Enterprise = 1 << 3, // ✓ Safe - new bit
};

// Step 2: Apply strict policies during development
#ifdef DEVELOPMENT_BUILD
template<class T>
using FeaturePolicy = alr::Evolve<T,
    EvolvePolicy::RequiredIfKnown_EnumFlagsStrict,
    EvolvePolicy::RequiredIfKnown_EnumFlagsStrict>;
#else
template<class T>
using FeaturePolicy = T;
#endif

struct Config {
    FeaturePolicy<FeatureFlags> features;
}

// Step 3: Test against v1.0 release
// If you accidentally did this:
//   enum class FeatureFlags {
//       Basic = 1 << 0,
//       Premium = 1 << 1,    // ❌ Conflict with Advanced!
//       Advanced = 1 << 2,   // ❌ Shifted!
//   };
// Policy violation would be triggered immediately during testing

// Step 4: Ship v2.0 with policies removed
// Policies compile away, zero overhead in production
```

This approach provides a **safety net during active development** without imposing any runtime cost in production, making it ideal for large refactorings that could result in breaking changes.

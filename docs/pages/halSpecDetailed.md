# Telco Voice JSON HAL Detailed Reference

See the [overview](halSpec.md). Telco Voice Manager consumes either `telcovoice_hal_schema_v1.json` or `telcovoice_hal_schema_v2.json`, selected by the product build and deployment. Each variant maps to Telco Voice Manager; do not combine their parameter contracts. JSON Pointers below refer to the selected variant.

## Envelope, Variant Selection, and Transport
Both schemas declare draft-07 and require:
| Field | Contract |
| --- | --- |
| `module` | String constant `voicehal`. |
| `version` | String constant `0.0.1` in **both** v1 and v2. |
| `action` | One of the eleven case-sensitive strings below. |
| `reqId` | Numeric string matching `^[0-9]+$`, not a JSON number. |

Use the deployed schema/build selection to distinguish variants. Telco Voice Manager configuration references an unversioned installed filename and port `20104`; neither the filename nor wire version identifies the checked-in variant by itself.

The shared library uses IPv4 TCP, with supplied server loopback binding. It is not HTTP or JSON-RPC 2.0. Messages have no robust length-prefix/delimiter framing in the supplied transport. TCP splitting/coalescing must not be assumed to preserve message boundaries.

## Action API Reference
| Action | Direction and payload contract |
| --- | --- |
| `getSchema` | Manager to vendor; definition describes retrieval without parameters. No action-specific root condition prohibits extra fields. |
| `getSchemaResponse` | Vendor to manager. Required `SchemaInfo` object with required `FilePath` string matching `^(.+)/([^/]+)$`; no other fields inside `SchemaInfo`. This returns a path, not inline schema data. |
| `getParameters` | Manager to vendor. Required nonempty unique `params` array. Each item requires `name` and read-list membership. |
| `getParametersResponse` | Vendor to manager. Read-list membership with required `name`, `type`, and `value`. |
| `setParameters` | Manager to vendor. Required nonempty unique `params` array. Each item requires `name`, `type`, `value`, and write-list membership. Atomicity is unspecified. |
| `subscribeEvent` | Manager to vendor. Required nonempty unique `params` array. Items require `name`, `notificationType`, and event-list membership. |
| `publishEvent` | Vendor to manager. Required nonempty unique `params` array. Items require `name`, `type`, and `value` and event-list membership. |
| `deleteObject` | Root requires nonempty unique `params` and item `name`, but both variants contain invalid empty deletion alternatives. No usable deletion target is specified. |
| `getActiveSubscriptions` | Enumerated only; no action-conditioned payload definition. |
| `getActiveSubscriptionsResponse` | Enumerated only; no action-conditioned payload definition. |
| `result` | Required `Result` object with required `Status`; no additional properties within that object. |

`Result.Status` permits `Success`, `Failed`, `Invalid Argument`, and `Not Supported`. The default is an annotation, not automatic insertion of a missing required field. C-library status is separate.

Array uniqueness compares complete elements rather than parameter names alone. Root and leaf additional-property rules differ; validate at the actual schema location.

## Complete Parameter-Contract Lookup
| Action index | v1 alternatives | v2 alternatives |
| --- | ---: | ---: |
| `/definitions/getParameterSupportedList/anyOf` | 148 | 501 |
| `/definitions/getParameterOptionalList/anyOf` | 248 | 399 |
| `/definitions/setParameterSupportedList/anyOf` | 58 | 215 |
| `/definitions/setParameterOptionalList/anyOf` | 179 | 315 |
| `/definitions/subscribeEventSupportedList/anyOf` | 1 | 1 |

v1/v2 contain 418/913 definitions respectively, including structural definitions, lists, and enums. Counts are references, not independent hardware capabilities.

For every referenced `/definitions/{key}`:
1. Use `description` for semantics, units, and access annotations, without inventing missing completion or failure guarantees.
2. Preserve `properties.name.pattern` exactly, including instance syntax and object trailing dots.
3. Preserve `properties.type.const` and independently enforce the JSON type and constraints under `properties.value`.
4. Resolve nested enum references and retain ranges, lengths, patterns, array/item restrictions, and additional-property policy.
5. Combine leaf restrictions with action-level required fields.
6. Determine action eligibility from the selected variant's lists. Read-side visibility is not permission to write.
7. Treat examples as annotations, not proof that a value passes validation.

v1 commonly organizes voice functionality through VoiceProfile/Line; v2 includes SIP, CallControl, POTS, and VoIPProfile structures. Do not translate paths or merge contracts implicitly.

### Initialization Parameter
`servicesVoiceHalInit` describes `Device.Services.VoiceHalInit` with textual type `boolean` and JSON Boolean value. Its existence does not by itself define the full manager startup sequence, thread safety, or idempotency. Verify action membership in the selected schema and the actual vendor handler.

### Event Parameter
Both variants list only `voiceServiceX_RDK_Firewall_Rule_Data` in the event index. Its path is `Device.Services.VoiceService.{index}.X_RDK_Firewall_Rule_Data`; it has textual type `string`, a constrained string value, and `notificationType` referencing the notification enum.

The description represents firewall rule data as enable/disable, protocol, port, and IP components. Enforce the exact schema pattern rather than accepting arbitrary CSV. That event does not establish subscription support for every line, call, or SIP status.

## Minimal Envelope Example
This demonstrates message shape, not verified runtime handler support:
```json
{"module":"voicehal","version":"0.0.1","action":"getSchema","reqId":"1"}
```

## Verified Schema Defects and Ambiguities
- Both variants contain `/definitions/deleteObject/anyOf: []`; draft-07 requires nonempty combinator arrays. Strict schema preflight can reject the document before testing any message.
- v2 references missing `#/definitions/voiceServiceCallLogSessionSessionRFactor`.
- No unresolved internal definition reference was found in v1.
- Active-subscription action names lack conditioned payload definitions.
- The identical module/version constants cannot negotiate or distinguish v1 and v2.

Do not silently fix these in generated tests. Obtain approved contract corrections and keep schema failures separate from vendor failures.

## Runtime, Security, and Recovery
A successful set result is not proof of successful SIP registration, call establishment, or diagnostic completion. Implementations must not log credentials, secrets, security-sensitive information, or other sensitive data. Restore changed voice configuration using an approved product procedure.

## Source References
- [Voice v1 schema](https://github.com/rdkcentral/telco-voice-manager/blob/main/hal_schema/telcovoice_hal_schema_v1.json)
- [Voice v2 schema](https://github.com/rdkcentral/telco-voice-manager/blob/main/hal_schema/telcovoice_hal_schema_v2.json)
- [Manager configuration](https://github.com/rdkcentral/telco-voice-manager/blob/main/config/telcovoice_manager_conf.json)

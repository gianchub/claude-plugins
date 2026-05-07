# Deserialization

## Scope

Untrusted-data deserialization is a top RCE class across multiple language ecosystems. This checklist covers Python pickle/yaml, Java ObjectInputStream / Jackson / SnakeYAML, PHP unserialize, .NET BinaryFormatter and friends, Ruby Marshal/YAML, and Node-side risks. The unifying principle: any deserialization API that can instantiate arbitrary types or invoke arbitrary methods on attacker-supplied class names is dangerous when fed untrusted input.

Modern alternatives are JSON (with explicit DTOs and validation), Protocol Buffers, and other structured formats that don't carry type-construction semantics.

## Python

### pickle / cPickle / dill / shelve

- **`pickle.loads`, `pickle.load`** — RCE via `__reduce__`. Any pickled byte stream from an untrusted source is RCE-class.
- **`shelve.open`** — Wraps pickle; same risk.
- **`dill`** — Extension of pickle; same risk and more.
- **No safe usage** — There is no "safe" mode for pickle of untrusted data. Replace with JSON / msgpack / protobuf.
- **Common contexts** — Caching layers (`memcached` / `redis` clients sometimes use pickle by default), session backends that pickle, message queues passing pickled payloads, ML model files (torch.save with pickle), data interchange in scientific pipelines.

### YAML

- **`yaml.load(data)`** — Without `Loader=yaml.SafeLoader`, allows construction of arbitrary Python objects via tags `!!python/object`, `!!python/object/apply` → RCE.
- **`yaml.load(data, Loader=SafeLoader)`** — Safe; only basic types.
- **`yaml.safe_load(data)`** — Safe; equivalent to SafeLoader.
- **`yaml.full_load(data)`** — Less dangerous than `load` but still allows arbitrary class construction; not safe for untrusted.
- **`yaml.unsafe_load(data)`** — Documented as unsafe; rarely justified.

### Marshal / shelve

- `marshal.loads`, `marshal.load` — Used internally by Python; not safe for untrusted, but exposure is rare in user code.

### XML

- **`xml.etree.ElementTree`** — Default behavior in modern Python disables external entities; older versions are vulnerable.
- **`lxml`** — Configure parser with `resolve_entities=False`, `no_network=True`, `dtd_validation=False`.
- **`xmltodict`** — Generally safe for entity expansion but verify behavior with attacker-controlled DTDs.

### JSON with custom decoders

- `json.loads` itself is safe; custom `object_hook` that instantiates classes by `__class__` field is not. Audit any object-hook code that calls user-controlled class names.

## Java / Kotlin

### `ObjectInputStream` / Native Java Serialization

- **`ObjectInputStream.readObject(...)`** — Classic Java deserialization RCE via gadget chains (Commons Collections, Spring, etc.). Even when the application doesn't define dangerous gadgets, the classpath does.
- **No safe deserialization filter** — Default behavior. Configure ObjectInputFilter (Java 9+) with allowlist of expected types.
- **JEP 290 ObjectInputFilter** — `setObjectInputFilter(...)` with strict allowlist; reject unexpected classes. Verify filter is set at the call site, not relying on JVM-wide filter.
- **Common gadget sources** — Commons-Collections 3.x (`InvokerTransformer`), Spring `BeanFactory`, Hibernate, JBoss; presence of these libs amplifies risk.
- **Migration** — JSON via Jackson with strict types; Protocol Buffers; never fix by adding custom serialization to dangerous types.

### Jackson

- **Default typing enabled** — `enableDefaultTyping(...)`, `activateDefaultTyping(...)`, `@JsonTypeInfo(use=JsonTypeInfo.Id.CLASS)` allow attacker to instruct Jackson to construct arbitrary classes via `@class` field. RCE via gadget chains (Spring, c3p0, etc.).
- **Polymorphic deserialization** — When using polymorphism, restrict to a closed set of subclasses via `@JsonSubTypes`. Use `JsonTypeInfo.Id.NAME` instead of `Id.CLASS`.
- **`JsonNode`** — Safe for parsing untrusted JSON; only converts to types when explicitly cast.
- **PolymorphicTypeValidator (PTV)** — Required when using polymorphic deserialization on untrusted data. Validates allowed types.

### SnakeYAML

- **`new Yaml().load(input)`** — Pre-2.0: same risk as Python yaml.load; constructs arbitrary classes via tags. SnakeYAML 2.0+ defaults to safe.
- **Safe construction** — `new Yaml(new SafeConstructor()).load(...)`.
- **Verify version** — Older Spring Boot bundles older SnakeYAML.

### XStream

- **`xstream.fromXML(...)`** — Default permissive deserialization; gadget-chain RCE.
- **Whitelist required** — `xstream.allowTypes(...)`, `xstream.allowTypeHierarchy(...)`. Configure for each XStream instance.

### XML

- **`DocumentBuilderFactory`** — Defaults vary by JVM and version. Set `setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)`, `setExpandEntityReferences(false)`, `setXIncludeAware(false)`.
- **`SAXParserFactory`, `XMLInputFactory`** — Same hardening required.
- **`Transformer`, `Validator`, `SchemaFactory`** — All can be vectors; harden each.

### Spring

- **Spring Messaging (`@SendTo`, `@MessageMapping`)** with default converters — May deserialize untrusted message bodies into types attacker controls.
- **`SpringEL` evaluation on user input** — Templating attacks (SSTI in Spring's expression language).

## PHP

### `unserialize`

- **`unserialize($input)`** — Object injection; instantiates classes; magic methods (`__wakeup`, `__destruct`, `__toString`) become attack surface.
- **Allowlist with `allowed_classes` option** — `unserialize($input, ['allowed_classes' => ['SafeClass']])` (PHP 7+); restricts class instantiation. Allowlist or `false` (no classes).
- **POP chains** — Common WordPress/Symfony/Laravel gadget chains; presence of these in `vendor/` amplifies risk.
- **Migration** — JSON via `json_decode($input, true)` (associative array); never use unserialize on untrusted.

### XML

- **`SimpleXMLElement`, `DOMDocument::loadXML`** — Default behavior may allow XXE; `LIBXML_NOENT` should not be set; consider explicitly disabling DTD with `LIBXML_DTDLOAD | LIBXML_DTDATTR` not set.

## .NET / C#

### `BinaryFormatter`

- **`BinaryFormatter.Deserialize`** — Notorious RCE class. Microsoft has officially deprecated and recommends removal.
- **`SoapFormatter`, `NetDataContractSerializer`, `LosFormatter`, `ObjectStateFormatter`** — All affected by the same class of issues.
- **Migration** — `System.Text.Json`, `Newtonsoft.Json` with strict settings, `MessagePack` (with strict resolver), Protocol Buffers.

### `Newtonsoft.Json` (Json.NET)

- **`TypeNameHandling`** — `Auto` / `Objects` / `All` / `Arrays` allow `$type` field to construct arbitrary classes → RCE.
- **`TypeNameHandling.None` (default)** — Safe; do not change.
- **`SerializationBinder`** — Custom binder restricting types; required if any TypeNameHandling != None used.

### `System.Text.Json`

- Safer by default; no type-name handling.
- **`JsonSerializer.Deserialize<T>`** with `T` being `object` or `dynamic` may surprise; use concrete DTOs.

### `JavaScriptSerializer`

- Older; similar TypeNameHandling-style risks with `SimpleTypeResolver`; deprecated.

### XML

- `XmlSerializer` — Generally safe (no arbitrary type construction without `KnownTypes`).
- `DataContractSerializer` — `KnownTypes` allowlist; verify for polymorphic.
- `XmlDocument`, `XmlReader` — Default `XmlReaderSettings` should set `DtdProcessing = Prohibit` for untrusted XML.

## Ruby

### Marshal

- **`Marshal.load`, `Marshal.restore`** — Object injection; gadget chains via Rails/PSafe libraries.
- **No safe mode** — Replace with JSON.

### YAML

- **`YAML.load(data)`** — In Psych < 4 or with permissive class allowlist, equivalent to Python yaml.load.
- **`YAML.safe_load(data)`** — Safer; only basic types.
- **`YAML.unsafe_load(data)`** — Explicitly unsafe; never on untrusted.
- **Psych 4+** — Default `load` is now safe (only basic types).
- **Older Rails** — May use unsafe load patterns.

### ERB / Liquid templates

- **ERB compiled from user input** — RCE; covered in injection.md SSTI section.
- **Liquid** — Sandboxed by design; SSTI-resistant but verify version (older versions had escapes).

### XML

- `Nokogiri` — Default behavior reasonable; verify `noent: false` not enabled.
- `REXML` — Old-style; check for entity expansion.

## Go

### `gob`

- **`gob.NewDecoder(reader).Decode(&v)`** — Decodes only into the specified target type; not arbitrary instantiation. Safer than pickle/Java but still: only decode into typed structures, never into `interface{}` for untrusted, and verify overall input size to prevent DoS.

### JSON

- **`json.Unmarshal`** — Safe; type-driven. Be careful with `interface{}` targets.
- **JSON unmarshaling and large payloads** — Memory DoS without size limits; use `http.MaxBytesReader`.

### XML

- `encoding/xml` — Default safe; no DTD support.

## Node.js / JavaScript

### JSON

- `JSON.parse(input)` — Safe by itself; *with a reviver function that instantiates classes*, becomes risky.
- **`JSON.parse(input, reviver)`** — Audit reviver for class construction or arbitrary code paths based on input keys.

### Other

- **`node-serialize`** — Insecure-by-design package; deserialization with code execution.
- **`serialize-javascript`** with `unsafe: true` — Allows function deserialization; never with untrusted.
- **YAML** — `js-yaml` safe modes: `yaml.load(input, { schema: yaml.SAFE_SCHEMA })`. Default schema also allows constructors; use safe schema.
- **`vm` module** — Not deserialization but related; running serialized JS code.

### XML

- `libxmljs` — Configure to disable entity processing.
- `xml2js` — Default safe; check options.

## Cross-Language: General Patterns

### Format Detection / Type Confusion

- **Same data accepted as multiple formats** — JSON parser accepts `--- a: 1` (looks like YAML) due to library leniency, then YAML parser also accepts.
- **Polyglot exploits** — File that's valid for two formats; one parser sees text, another sees code.

### Size and Complexity DoS

- **Billion laughs** — XML / SVG entity expansion DoS.
- **Quadratic blowup in JSON** — Deeply nested objects exhausting parser stack/memory.
- **Hash-collision DoS** — Object with thousands of keys hash-colliding to slow lookup. Modern parsers / runtimes mitigate; legacy code may not.

### Schema Validation

- **JSON Schema** validation — Validates structure; doesn't prevent deserialization-class issues for typed deserialization.
- **OpenAPI / typed binders** — Convert to typed objects; verify mass-assignment-style protections (covered in authorization.md).

## Recommendation Patterns

- Use JSON (or another format without type-construction semantics) and explicit DTOs for all untrusted deserialization.
- For Java, configure `ObjectInputFilter` strictly when native serialization can't be removed; long-term, remove.
- For .NET, eliminate `BinaryFormatter` per Microsoft guidance; migrate to `System.Text.Json` with concrete types.
- For Python, never use `pickle.loads` or `yaml.load` on untrusted; use `yaml.safe_load`, `json.loads`, or `msgpack` (which is safe in default mode).
- For PHP, replace `unserialize` with `json_decode`; if unserialize must remain, set `allowed_classes => false` or a strict allowlist.
- Verify size/complexity limits at parser configuration.
- Audit dependency updates: deserialization-related CVEs are frequent and high-severity (Jackson, SnakeYAML, Newtonsoft, etc.).

# Custom Structure Instances

A Custom Structure Definition (CSD) lets an agency define a brand new kind of
maintainable artefact that does not exist in the SDMX Information Model — a
pivot table layout, a glossary, a report template. Because the shape of those
artefacts is invented by the agency rather than fixed by SDMX, the standard
SDMX-ML schemas cannot describe them.

What the standard schemas do describe is the *definition* itself
(`str:CustomStructureDefinitions`, typed by `CustomStructureDefinitionType` in
`SDMXStructureCustomStructure.xsd`) and the *container* which holds the
instances (`str:CustomStructures`, typed by `CustomStructuresType`). That
container is deliberately open: its content model is a single

```xml
<xs:any namespace="##other" processContents="lax" minOccurs="0" maxOccurs="unbounded"/>
```

so the agency's own elements are allowed through without being checked.

Everything else has to come from a second schema, **derived from the CSD**.
Given a definition that declares a `dataflow` property and a repeating `rows`
property, you can work out mechanically that instances must carry a `dataflow`
element holding a dataflow URN and may carry any number of `rows` elements. That
derived schema is what the `*-1.0.0.xsd` files in the samples folder are. They
are not part of the SDMX distribution and never will be: there is one per CSD
version, they are produced by software from the CSD, and they are only
meaningful to systems that know about that particular CSD.

Each instance is an element **in a namespace which is the URN of the definition
it conforms to**, with the element name being the definition's `sdmxClassName`:

```xml
<str:CustomStructures>
    <pt:PivotTable urn="urn:sdmx:org.sdmx.infomodel.csd.imf.PivotTable=OECD:POP_SEX_AGE(1.0.0)"
            agencyID="OECD" id="POP_SEX_AGE" version="1.0.0">
        …
    </pt:PivotTable>
</str:CustomStructures>
```

where `pt` is bound to
`urn:sdmx:org.sdmx.infomodel.csd.CustomStructureDefinition=IMF:PIVOT_TABLE(1.0.0)`.
This gives a two-level validation model, which is intentional:

| Validating with | Checks |
| --- | --- |
| The standard schema set alone (`SDMXMessage.xsd`) | The message, the definitions in full, and the well-formedness of each instance. Custom content passes untouched. |
| The standard schema set **plus** the generated schemas | The above, and the custom content of every instance whose definition you hold. |

An instance of a CSD you have never seen is therefore never rejected — it is
simply not checked in depth, because `processContents="lax"` skips an element
whose namespace the validator has no schema for, and the dangling reference is
caught later by referential integrity checking. This mirrors the open
`CustomStructureInstanceType` used by SDMX-JSON for the same purpose.

# Generating Schema for Custom Instances

The rules below turn one CSD into one XML Schema. They are deterministic: two
implementations following them produce equivalent schemas.

Throughout, `common:` stands for the prefix bound to
`http://www.sdmx.org/resources/sdmxml/schemas/v3_2/common`. Every example is
taken from the samples folder for custom structure definitions: the running
example builds up `pivot_table_1.0.0.xsd` from `pivot_table_csd.xml`, piece by
piece, and where the pivot table does not exercise a rule the glossary
(`custom_item_scheme_csd.xml` → `glossary_1.0.0.xsd`) is used instead.
Annotations are trimmed from the fragments to keep them readable.

## 1. What the generated schema validates

Generate **one schema per CSD version**, and have it validate **one instance
element** — a single member of the message's `str:CustomStructures` container —
not a whole message. Combining the instance schemas into a schema set that
validates a message is a separate step, covered in the last section.

```xml
<xs:schema targetNamespace="urn:sdmx:org.sdmx.infomodel.csd.CustomStructureDefinition=IMF:PIVOT_TABLE(1.0.0)"
            xmlns="urn:sdmx:org.sdmx.infomodel.csd.CustomStructureDefinition=IMF:PIVOT_TABLE(1.0.0)"
            xmlns:xs="http://www.w3.org/2001/XMLSchema"
            xmlns:common="http://www.sdmx.org/resources/sdmxml/schemas/v3_2/common"
            elementFormDefault="qualified">

    <xs:import namespace="http://www.sdmx.org/resources/sdmxml/schemas/v3_2/common"
            schemaLocation="https://xml.sdmx.org/3.2/SDMXCommon.xsd"/>

    <xs:element name="PivotTable" type="PivotTableType"/>

    <xs:simpleType name="PivotTableUrnType"> … </xs:simpleType>          <!-- section 2 -->
    <xs:complexType abstract="true" name="PivotTableBaseType"> … </xs:complexType>
    <xs:complexType name="PivotTableType"> … </xs:complexType>
    <xs:complexType abstract="true" name="RowColTypeBaseType"> … </xs:complexType>  <!-- section 3 -->
    <xs:complexType name="RowColTypeType"> … </xs:complexType>
    <xs:complexType name="SliceTypeType"> … </xs:complexType>            <!-- section 4 -->
</xs:schema>
```

**targetNamespace** — the URN of the CSD the schema is generated from. Also bind
it as the default namespace so that generated type names need no prefix. The
namespace is the dispatch key: because it contains the version of the
definition, instances of different versions of the same definition can appear in
one message and each is validated by its own generated schema.

**elementFormDefault** — fixed value `qualified`. Every property element is in
the CSD's namespace, so instances qualify them (`pt:dataflow`, `gl:definition`).

**xs:import** — the SDMX common namespace, which supplies the base types
(`common:MaintainableType`, `common:NameableType`, …), the reference types and
the reusable `common:Annotations`, `common:Link`, `common:Name` and
`common:Description` elements. Point `schemaLocation` at the standard
schemas locaiton or local resource.

**xs:element** — one global element, named by the `sdmxClassName` of the CSD.
This is the element that appears inside `str:CustomStructures`, and it is the
only global element the schema declares.

The rest of the schema is generated types. Their names are derived
predictably from the CSD, so check that none of them collides with another:

| Generated component | Name | From |
| --- | --- | --- |
| Global element | `PivotTable` | the CSD's `sdmxClassName` |
| Instance URN simple type | `PivotTableUrnType` | `sdmxClassName` + `UrnType` |
| Abstract root complex type | `PivotTableBaseType` | `sdmxClassName` + `BaseType` |
| Root complex type | `PivotTableType` | `sdmxClassName` + `Type` |
| Abstract custom type | `RowColTypeBaseType` | custom type `id` + `BaseType` |
| Custom type | `RowColTypeType` | custom type `id` + `Type` |

`RowColTypeType` reads awkwardly, but the `Type` suffix is applied
unconditionally so that the rule stays mechanical and the names of a custom type
called `Row` and one called `RowType` cannot collide. Neither suffix comes from
the modeller, so check that `<id>Type` and `<id>BaseType` do not collide with
another custom type `id` in the CSD, since those are the names a modeller
controls.

The JSON rules name their equivalent of `<id>BaseType` `Abstract<id>`, but the
two are not the same construct and are deliberately not named alike: the JSON
abstract definition exists only where one custom type extends another, whereas
`<id>BaseType` exists wherever a custom type is identifiable (see section 3).

## 2. The Root Type

The root of the instance is always maintainable, so it is generated as a pair of
complex types: an abstract `<sdmxClassName>BaseType` which **extends**
`common:MaintainableType` to add the declared content, and a concrete
`<sdmxClassName>Type` which **restricts** that base type to narrow the `urn`
attribute. The pair is needed because XML Schema cannot restrict an attribute
and extend the content model in one step.

```xml
<xs:simpleType name="PivotTableUrnType">
    <xs:restriction base="common:CustomInstanceUrnType">
        <xs:pattern value=".+\.csd\.imf\.PivotTable=.+"/>
    </xs:restriction>
</xs:simpleType>

<xs:complexType abstract="true" name="PivotTableBaseType">
    <xs:complexContent>
        <xs:extension base="common:MaintainableType">
            <xs:sequence>
                <xs:element name="CustomStructureDefinition" type="common:CustomStructureDefinitionReferenceType"
                        fixed="urn:sdmx:org.sdmx.infomodel.csd.CustomStructureDefinition=IMF:PIVOT_TABLE(1.0.0)"/>
                <xs:element name="dataflow" type="common:DataflowReferenceType"/>
                <xs:element name="rows" type="RowColTypeType" minOccurs="0" maxOccurs="unbounded"/>
                <xs:element name="cols" type="RowColTypeType" minOccurs="0" maxOccurs="unbounded"/>
                <xs:element name="slice" type="SliceTypeType" minOccurs="0" maxOccurs="unbounded"/>
            </xs:sequence>
        </xs:extension>
    </xs:complexContent>
</xs:complexType>

<xs:complexType name="PivotTableType">
    <xs:complexContent>
        <xs:restriction base="PivotTableBaseType">
            <xs:sequence>
                <xs:element ref="common:Annotations" minOccurs="0"/>
                <xs:element ref="common:Link" minOccurs="0" maxOccurs="unbounded"/>
                <xs:element ref="common:Name" maxOccurs="unbounded"/>
                <xs:element ref="common:Description" minOccurs="0" maxOccurs="unbounded"/>
                <xs:element name="CustomStructureDefinition" type="common:CustomStructureDefinitionReferenceType"
                        fixed="urn:sdmx:org.sdmx.infomodel.csd.CustomStructureDefinition=IMF:PIVOT_TABLE(1.0.0)"/>
                <xs:element name="dataflow" type="common:DataflowReferenceType"/>
                <xs:element name="rows" type="RowColTypeType" minOccurs="0" maxOccurs="unbounded"/>
                <xs:element name="cols" type="RowColTypeType" minOccurs="0" maxOccurs="unbounded"/>
                <xs:element name="slice" type="SliceTypeType" minOccurs="0" maxOccurs="unbounded"/>
            </xs:sequence>
            <xs:attribute name="urn" type="PivotTableUrnType" use="optional"/>
        </xs:restriction>
    </xs:complexContent>
</xs:complexType>
```

**The URN simple type** restricts `common:CustomInstanceUrnType` to the class of
this definition. The pattern is `.+\.csd\.<agency>\.<sdmxClassName>=.+`, where
`<agency>` is the `agencyID` of the **definition** lower-cased (a nested agency
keeps its `.` separators). Note that this is the agency of the definition, not of
the instance: in the sample, the class part is `csd.imf.PivotTable` because IMF
maintains the definition, while the instance itself is maintained by OECD and so
its URN is `urn:sdmx:org.sdmx.infomodel.csd.imf.PivotTable=OECD:POP_SEX_AGE(1.0.0)`.
The version of the definition does not appear in the instance URN — it is
carried by the instance's reference to its definition — so it does not appear in
the pattern either.

**The abstract type** extends `common:MaintainableType`, which brings in
`common:Annotations`, `common:Link`, `common:Name`, `common:Description` and the
`id`, `agencyID`, `version`, `urn`, `uri`, `validFrom`, `validTo`,
`isExternalReference` and `isPartialLanguage` attributes. To that it adds, in a
sequence:

1. `CustomStructureDefinition`, typed `common:CustomStructureDefinitionReferenceType`
   and **fixed** to the URN of the definition this schema was generated from.
   The element is mandatory, and fixing it guarantees the reference agrees with
   the namespace the instance element is in. The identifier
   `CustomStructureDefinition` is reserved by `PropertyType` for this purpose,
   so it can never clash with a declared property.
2. the reserved `items` element, when `base="ItemScheme"` (see below).
3. one element per declared `Property`, in declaration order, following the
   rules in section 5.

**The concrete type** restricts the abstract type. A restriction must restate the
whole content model, so the four inherited elements are repeated ahead of the
generated ones, exactly as `MaintainableBaseType` declares them, and the
generated ones are repeated unchanged. The only thing that actually changes is
the `urn` attribute, which is narrowed from `common:MaintainableUrnType` to the
generated `PivotTableUrnType`. Attributes which are not restated are inherited
unchanged.

### base="ItemScheme"

When the CSD sets `base="ItemScheme"` the definition must declare a property
identified `items` whose representation is a `CustomTypeReference` to a custom
type extending `Nameable`. That property plays the item role. Two things change
in the generated abstract type: the `items` element is emitted first, ahead of the
other declared properties, and the `isPartial` attribute of item schemes is
added.

```xml
<xs:complexType abstract="true" name="GlossaryBaseType">
    <xs:complexContent>
        <xs:extension base="common:MaintainableType">
            <xs:sequence>
                <xs:element name="CustomStructureDefinition" type="common:CustomStructureDefinitionReferenceType"
                        fixed="urn:sdmx:org.sdmx.infomodel.csd.CustomStructureDefinition=IMF:GLOSSARY(1.0.0)"/>
                <xs:element name="items" type="TermTypeType" minOccurs="0" maxOccurs="unbounded"/>
                <xs:element name="subjectArea" type="common:ConceptReferenceType" minOccurs="0"/>
            </xs:sequence>
            <xs:attribute name="isPartial" type="xs:boolean" use="optional" default="false"/>
        </xs:extension>
    </xs:complexContent>
</xs:complexType>
```

Apart from those two additions the type is generated exactly as a `Maintainable`
one: `items` is still an ordinary property as far as the rules in section 5 are
concerned, and `subjectArea` shows that an item scheme based definition may
declare properties alongside its items.

## 3. Custom Types

Each `CustomType` in the definition becomes a complex type, accompanied by an
abstract `<id>BaseType` when the custom type extends `Identifiable` or
`Nameable`. The base type carries the inherited properties, and the concrete
type extends it with the custom type's own properties.

The split is needed for the same reason as at the root:
`common:IdentifiableType` declares `id` as optional, and the id of a custom
object is needed to generate its URN, so it has to be restricted to required
before the type's own properties can be extended in, and XML Schema cannot
restrict an attribute and extend the content model in one step.

Note that this is a wider rule than the JSON one, which generates its
`Abstract<id>` only for a custom type which another custom type extends. In XML
Schema an extending type derives from the concrete parent type directly (section
4), so extension by itself never calls for a base type; the base type is instead
what makes the id mandatory. `TermTypeBaseType` in the glossary sample shows
this — nothing extends `TermType`, and it still has one.

```xml
<xs:complexType abstract="true" name="RowColTypeBaseType">
    <xs:complexContent>
        <xs:restriction base="common:NameableType">
            <xs:sequence>
                <xs:element ref="common:Annotations" minOccurs="0"/>
                <xs:element ref="common:Link" minOccurs="0" maxOccurs="unbounded"/>
                <xs:element ref="common:Name" maxOccurs="unbounded"/>
                <xs:element ref="common:Description" minOccurs="0" maxOccurs="unbounded"/>
            </xs:sequence>
            <xs:attribute name="id" type="common:IDType" use="required"/>
        </xs:restriction>
    </xs:complexContent>
</xs:complexType>

<xs:complexType name="RowColTypeType">
    <xs:complexContent>
        <xs:extension base="RowColTypeBaseType">
            <xs:sequence>
                <xs:element name="level" minOccurs="0"> … </xs:element>
                <xs:element name="dimension" type="common:IDType"/>
                <xs:choice minOccurs="0">
                    <xs:element name="headingText" type="xs:string"/>
                    <xs:element name="headingCode" type="common:CodeReferenceType"/>
                </xs:choice>
            </xs:sequence>
        </xs:extension>
    </xs:complexContent>
</xs:complexType>
```

Pick the base from the custom type's `extends` attribute:

| `extends` | Generate an abstract `<id>BaseType` | `<id>Type` extends |
| --- | --- | --- |
| `Annotatable` (or absent, with no `sdmxClassName`) | not needed | `common:AnnotableType` |
| `Identifiable` (or absent, with an `sdmxClassName`) | restricting `common:IdentifiableType`, `id` required | `<id>BaseType` |
| `Nameable` | restricting `common:NameableType`, `id` required | `<id>BaseType` |
| `extendsType="X"` | not needed, see section 4 | `XType` |

An `Annotatable` custom type has no id and therefore no URN, so nothing needs
restricting and the type extends `common:AnnotableType` directly. `ItemScheme`
is not available to a custom type; it is a base for the definition itself only.

The `sdmxClassName` of a custom type plays no part in the generated schema. It
is the class used when the URN of an object of that type is generated —
`urn:sdmx:org.sdmx.infomodel.csd.imf.PivotTableRow=OECD:POP_SEX_AGE(1.0.0).SEX_ROW`
for the `SEX_ROW` object in the sample — and each type in an extension chain
must declare its own so that those URNs stay unique. Nested objects inherit the
optional `urn` attribute from `common:IdentifiableType` and may report it, as
the samples do; a generator may additionally restrict that attribute to the
type's own class in the same way section 2 does for the root, but the standard
type is sufficient.

### Recursion

A custom type may reference itself or its container. A named complex type
referenced by `type=` handles this naturally; just make sure your generator does
not try to inline types and loop forever. The glossary's `TermType` declares a
`children` property whose representation is `TermType`, which becomes:

```xml
<xs:complexType name="TermTypeType">
    <xs:complexContent>
        <xs:extension base="TermTypeBaseType">
            <xs:sequence>
                <xs:element name="definition" type="common:TextType" maxOccurs="unbounded"/>
                <xs:element name="children" type="TermTypeType" minOccurs="0" maxOccurs="unbounded"/>
            </xs:sequence>
        </xs:extension>
    </xs:complexContent>
</xs:complexType>
```

## 4. Extension Chains

When a custom type carries `extendsType`, it extends a sibling custom type
rather than an SDMX base. The generated type extends the sibling's **concrete**
type directly, adds only its own properties, and needs no base type of its own —
the id was already made required further up the chain.

```xml
<xs:complexType name="SliceTypeType">
    <xs:complexContent>
        <xs:extension base="RowColTypeType">
            <xs:sequence>
                <xs:element name="position" minOccurs="0">
                    <xs:simpleType>
                        <xs:restriction base="xs:int">
                            <xs:minInclusive value="0"/>
                        </xs:restriction>
                    </xs:simpleType>
                </xs:element>
            </xs:sequence>
        </xs:extension>
    </xs:complexContent>
</xs:complexType>
```

Extension in XML Schema appends to the content model, so the subtype's own
properties always follow the inherited ones in the instance, and everything the
parent declared — including any `xs:choice` generated from a mutually exclusive
set — applies unchanged. Extension is additive only: a subtype must not redefine
or restrict an inherited property, and extension chains must be acyclic, both of
which the CSD itself already requires.

This is one place where the XML rules are simpler than their JSON counterparts.
There, the extending type cannot reference the parent type — a closed JSON
Schema object would reject the added properties — so the shared properties have
to be hoisted into an `Abstract<parent>` definition which both types reference.
In XML Schema `SliceTypeType` derives from `RowColTypeType` itself, so
`RowColTypeBaseType` is not what makes the extension work; it is there because
`RowColType` extends `Nameable`, exactly as `TermTypeBaseType` is.

## 5. Properties

Each declared `Property` becomes one element declaration, whose name is the
property `id`. Because the identifier becomes an element name it is typed
`common:NCNameIDType` in the CSD, so it is always a legal name; the identifiers
`id`, `name`, `description`, `items` and `CustomStructureDefinition` are
reserved and cannot be declared.

Properties are emitted **in declaration order** into an `xs:sequence`, and that
order is significant: an instance must report its properties in the order the
definition declares them. This is a real difference from the JSON rules, where
object members are unordered and the order of properties in the CSD carries no
meaning.

### 5a. Cardinality

`minOccurs` and `maxOccurs` map straight onto their XML Schema equivalents, but
their **defaults differ from the XML Schema defaults** and this is the one place
a generator is likely to get it wrong:

| Attribute | CSD default | XML Schema default |
| --- | --- | --- |
| `minOccurs` | 1 | 1 |
| `maxOccurs` | **unbounded** | **1** |

`maxOccurs` defaults to unbounded so that the absence of the attribute carries
the same meaning in every format — the JSON formats have no literal for
`unbounded` and express it by omitting the attribute. A single valued property
must therefore state `maxOccurs="1"` explicitly, and a generator must never let
an omitted `maxOccurs` fall through to the XML Schema default of 1.

| Definition says | Reads as | Generate |
| --- | --- | --- |
| `<str:Property id="dataflow" maxOccurs="1"/>` | 1..1 | no `minOccurs`, no `maxOccurs` |
| `<str:Property id="level" minOccurs="0" maxOccurs="1"/>` | 0..1 | `minOccurs="0"` |
| `<str:Property id="rows" minOccurs="0" maxOccurs="unbounded"/>` | 0..* | `minOccurs="0" maxOccurs="unbounded"` |
| `<str:Property id="definition" maxOccurs="unbounded"/>` | 1..* | `maxOccurs="unbounded"` |

```xml
<xs:element name="dataflow" type="common:DataflowReferenceType"/>
<xs:element name="level" minOccurs="0"> … </xs:element>
<xs:element name="rows" type="RowColTypeType" minOccurs="0" maxOccurs="unbounded"/>
<xs:element name="definition" type="common:TextType" maxOccurs="unbounded"/>
```

Emit `minOccurs` only when it is not 1 and `maxOccurs` only when it is not 1,
and copy any other stated bound through as it stands (`minOccurs="2"`,
`maxOccurs="5"`). A property which belongs to a `MutuallyExclusive` set is
effectively optional whatever its `minOccurs` says; see section 5c.

### 5b. Representation

A property carries exactly one representation — the CSD makes the choice of
`ValueFormat`, `Reference`, `IndirectReference` and `CustomTypeReference`
mandatory — and it decides the element's type.

**`ValueFormat`** → a simple typed value. The `textType` attribute, which
defaults to `String`, selects the type, and the facets restrict it:

| `textType` | Type |
| --- | --- |
| `String` (the default), `XHTML` | `xs:string`, `common:XHTMLType` |
| `Alpha`, `AlphaNumeric`, `Numeric` | `common:AlphaType`, `common:AlphaNumericType`, `common:NumericType` |
| `BigInteger`, `Integer`, `Long`, `Short`, `Count` | `xs:integer`, `xs:int`, `xs:long`, `xs:short`, `xs:integer` |
| `Decimal`, `Float`, `Double` | `xs:decimal`, `xs:float`, `xs:double` |
| `InclusiveValueRange`, `ExclusiveValueRange`, `Incremental` | `xs:decimal` |
| `Boolean`, `URI`, `Duration` | `xs:boolean`, `xs:anyURI`, `xs:duration` |
| `DateTime`, `Time`, `Month`, `MonthDay`, `Day` | `xs:dateTime`, `xs:time`, `xs:gMonth`, `xs:gMonthDay`, `xs:gDay` |
| `GregorianYear`, `GregorianYearMonth`, `GregorianDay` | `xs:gYear`, `xs:gYearMonth`, `xs:date` |
| `ObservationalTimePeriod`, `StandardTimePeriod`, `BasicTimePeriod`, `GregorianTimePeriod`, `TimeRange` | the `common:` type of the same name, e.g. `common:ObservationalTimePeriodType` |
| `ReportingTimePeriod`, `ReportingYear`, `ReportingSemester`, `ReportingTrimester`, `ReportingQuarter`, `ReportingMonth`, `ReportingWeek`, `ReportingDay` | the `common:` type of the same name, e.g. `common:ReportingQuarterType` |
| anything else | `xs:string` |

The facets are the standard SDMX ones and translate directly, because the value
is typed:

| Facet | Generate |
| --- | --- |
| `minLength` / `maxLength` | `xs:minLength` / `xs:maxLength` |
| `pattern` | `xs:pattern` (an XML Schema pattern is anchored implicitly — it must match the whole value — so do not add `^` and `$`) |
| `minValue` / `maxValue` | `xs:minInclusive` / `xs:maxInclusive` |
| `decimals` | `xs:fractionDigits` |

A property with facets gets an anonymous simple type, one without them is typed
by reference:

```xml
<str:Property id="level" minOccurs="0" maxOccurs="1">
    <str:ValueFormat textType="Integer" minValue="0"/>
</str:Property>
<str:Property id="headingText" minOccurs="0" maxOccurs="1">
    <str:ValueFormat textType="String"/>
</str:Property>
```
```xml
<xs:element name="level" minOccurs="0">
    <xs:simpleType>
        <xs:restriction base="xs:int">
            <xs:minInclusive value="0"/>
        </xs:restriction>
    </xs:simpleType>
</xs:element>
<xs:element name="headingText" type="xs:string"/>
```

Note that the JSON rules have to express `minValue="0"` as the regular
expression `^(0|[1-9][0-9]*)$`, because a JSON custom instance reports every
value in its lexical form as a string. XML Schema types the element, so the
bound is stated natively and is checked as a number.

**`ValueFormat` with `isMultiLingual="true"`** → `common:TextType`, which is a
string carrying an `xml:lang` attribute. Such a property is normally left
unbounded so that it can be reported once per language:

```xml
<str:Property id="definition" maxOccurs="unbounded">
    <str:ValueFormat textType="String" isMultiLingual="true"/>
</str:Property>
```
```xml
<xs:element name="definition" type="common:TextType" maxOccurs="unbounded"/>
```

which matches instance content of:

```xml
<gl:definition xml:lang="en">The total monetary value of all final goods and services …</gl:definition>
<gl:definition xml:lang="fr">La valeur monétaire totale de tous les biens et services finaux …</gl:definition>
```

The cardinality of a multilingual property is interpreted per language: the
`definition` above is mandatory, meaning one value must be present, not one per
language. XML Schema cannot check that the language of two values differs, so
that is left to the system.

**`CustomTypeReference`** → the complex type you generated for it in section 3:

```xml
<str:Property id="rows" minOccurs="0" maxOccurs="unbounded">
    <str:CustomTypeReference type="RowColType"/>
</str:Property>
```
```xml
<xs:element name="rows" type="RowColTypeType" minOccurs="0" maxOccurs="unbounded"/>
```

**`Reference`** → the value is the URN of the referenced artefact. Pick the
tightest type in the common namespace:

| `Reference` holds | Generate |
| --- | --- |
| one `Target` with a `class` which has a dedicated reference type | that type, e.g. `common:DataflowReferenceType` |
| several `Target` elements | an anonymous `xs:union` of their reference types |
| a `Target` with a `csd` attribute | `common:CustomInstanceUrnReferenceType`, restricted by pattern to the class and agency of that definition where it is known (as in section 2) |
| no `Target`, or `class="Any"` | `common:UrnReferenceType` |

```xml
<str:Property id="dataflow" maxOccurs="1">
    <str:Reference><str:Target class="Dataflow"/></str:Reference>
</str:Property>
```
```xml
<xs:element name="dataflow" type="common:DataflowReferenceType"/>
```

and, for a property that may point at either of two classes:

```xml
<str:Property id="source" maxOccurs="1">
    <str:Reference>
        <str:Target class="Dataflow"/>
        <str:Target class="DataStructure"/>
    </str:Reference>
</str:Property>
```
```xml
<xs:element name="source">
    <xs:simpleType>
        <xs:union memberTypes="common:DataflowReferenceType common:DataStructureReferenceType"/>
    </xs:simpleType>
</xs:element>
```

Use the `…ReferenceType` family rather than the `…UrnType` family: the reference
types accept wildcarded and late bound versions as well as absolute ones.

**`IndirectReference`** → the value is an identifier, not a URN, so the element
is typed `common:IDType`:

```xml
<str:Property id="dimension" maxOccurs="1">
    <str:IndirectReference targetClass="Dimension" context="dataflow"/>
</str:Property>
```
```xml
<xs:element name="dimension" type="common:IDType"/>
```

Whether that identifier actually resolves against the artefact named by
`context` — here, whether `AGE` really is a dimension of the dataflow in the
instance's `dataflow` element — cannot be expressed in XML Schema. Leave it to
the system.

### 5c. Mutually exclusive sets

Each `MutuallyExclusive` set becomes one `xs:choice`, which is the natural XML
construct for the constraint and lets schema validation enforce it directly:

```xml
<str:MutuallyExclusive>
    <str:Member property="headingText"/>
    <str:Member property="headingCode"/>
</str:MutuallyExclusive>
```
```xml
<xs:choice minOccurs="0">
    <xs:element name="headingText" type="xs:string"/>
    <xs:element name="headingCode" type="common:CodeReferenceType"/>
</xs:choice>
```

The members are removed from the enclosing sequence and generated inside the
choice instead, at the position where the first of them was declared. The choice
itself takes `minOccurs="0"`, because a member of a mutually exclusive set is
effectively optional whatever its own `minOccurs` says; each member keeps its
own `maxOccurs`. A set of three or more members is one choice with three or more
branches — unlike the JSON rules, which need one clause per pair.

Because the members move into the choice, the order in which the definition
declares them still determines the order of the branches, but only one branch
can be present in an instance so no ordering question arises for the reader.

## 6. What is deliberately not generated

- **Consistency of the `urn` attribute.** The URN of a row, a glossary term and
  so on is derived from the instance, the class name of its type and the id path
  of its identifiable ancestors. The generated schema types the root instance
  URN (section 2) but does not check that a reported URN agrees with the value
  the derivation would produce, nor that a nested object's URN matches its
  position. The attribute is optional throughout.
- **Uniqueness of ids** among sibling identifiable objects. XML Schema could
  express some of this with `xs:unique`, but not the general case of a recursive
  hierarchy, so it is left to the system throughout for consistency.
- **Resolution of indirect references**, and of references generally: that the
  URN in `dataflow` names a dataflow which exists, and that `dimension` names one
  of its dimensions, is referential integrity checking, not schema validation.
- **`SentinelValue`**, `isSequence` and the sequence facets (`interval`,
  `startValue`, `endValue`, `timeInterval`, `startTime`, `endTime`) of a
  `ValueFormat`. None of them has an XML Schema equivalent; a generator may emit
  them into an `xs:annotation` for documentation, and the system enforces them.

# Composing the validation schema set

The generated schemas each validate one instance element. To validate a whole
message you need a small wrapper schema which pulls in the standard SDMX schemas
and the generated schema of every definition the system knows about.
`csd_validation.xsd` in the samples folder is a worked example, covering two
CSDs:

```xml
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema">
    <xs:import namespace="http://www.sdmx.org/resources/sdmxml/schemas/v3_2/message"
            schemaLocation="../../schemas/SDMXMessage.xsd"/>
    <xs:import namespace="urn:sdmx:org.sdmx.infomodel.csd.CustomStructureDefinition=IMF:PIVOT_TABLE(1.0.0)"
            schemaLocation="pivot_table_1.0.0.xsd"/>
    <xs:import namespace="urn:sdmx:org.sdmx.infomodel.csd.CustomStructureDefinition=IMF:GLOSSARY(1.0.0)"
            schemaLocation="glossary_1.0.0.xsd"/>
</xs:schema>
```

A system composes this file at runtime from the CSDs it currently holds, and
regenerates it whenever that set changes. It is the XML counterpart of
`csd_validation-schema.json` in the SDMX-JSON samples, which does the same job
with one `if`/`then` per known CSD.

Four points are worth understanding before you write your own:

- **The wrapper has no target namespace of its own.** It declares nothing; it
  exists only to make one document out of several namespaces, so that a
  validator loading it has all of them available at once.
- **Dispatch is by namespace, and it is the validator's own.** No conditional
  logic is needed. The instance element carries the namespace of its definition,
  the schema set contains a declaration for that element in that namespace, and
  the validator matches the two.
- **Unknown definitions are allowed through.** An instance whose namespace is in
  no import has no declaration to match, and `processContents="lax"` on the
  `CustomStructures` wildcard means the validator skips it rather than
  reporting an error. This is the lax behaviour, and it is a feature: you do not
  need every agency's CSDs to process a message.
- **The version is part of the key.** Because the namespace URN includes
  `(1.0.0)`, instances of different versions of the same definition can appear
  in one message, each validated by its own generated schema.

An instance document names the schemas it should be validated against in the
usual way. For a single definition this can be done with `xsi:schemaLocation`
alone, which is what the samples do:

```xml
<mes:Structure xmlns:pt="urn:sdmx:org.sdmx.infomodel.csd.CustomStructureDefinition=IMF:PIVOT_TABLE(1.0.0)"
        xsi:schemaLocation="http://www.sdmx.org/resources/sdmxml/schemas/v3_2/message ../../schemas/SDMXMessage.xsd
                            urn:sdmx:org.sdmx.infomodel.csd.CustomStructureDefinition=IMF:PIVOT_TABLE(1.0.0) pivot_table_1.0.0.xsd">
```

Because dispatch is by namespace, the `CustomStructureDefinition` element of the
instance plays no part in choosing the schema — but it is mandatory, and the
generated schema fixes it to the URN of the definition, so an instance in the
right namespace which references a different definition is rejected.

# Examples

Two custom structure definitions, one of each `base`, with the derived schema
and a conforming instance for each. They are the XML equivalents of the samples
in `docs/structure_message` in the
[sdmx-json](https://github.com/sdmx-twg/sdmx-json) repository, so the two
formats can be compared side by side.

| File | What it is |
| --- | --- |
| `pivot_table_csd.xml` | A CSD with `base="Maintainable"`, defining a pivot table layout over a dataflow. Shows all four property representations, an `extendsType` chain (`SliceType` extends `RowColType`) and a `MutuallyExclusive` set. |
| `pivot_table_1.0.0.xsd` | The schema derived from it, following the rules above. |
| `pivot_table_instance.xml` | A pivot table maintained by OECD, conforming to the IMF definition. |
| `custom_item_scheme_csd.xml` | A CSD with `base="ItemScheme"`, defining a glossary. Shows the reserved `items` property, a multilingual property, and a custom type that references itself to build a hierarchy. |
| `glossary_1.0.0.xsd` | The schema derived from it. |
| `glossary_instance.xml` | A glossary maintained by ECB, conforming to the IMF definition. |
| `csd_validation.xsd` | The wrapper described in the previous section, covering both definitions. |

All seven are in the `samples/Custom Structure Definition` folder. To validate
the two instances in full, validate them against `csd_validation.xsd` with any
XML Schema 1.0 validator; the `../../schemas/` locations resolve to the standard
schemas in this repository. Validating them against `SDMXMessage.xsd` alone
checks the message and the maintainable parts only, and passes over the custom
content — a useful way to see the difference the generated schemas make.

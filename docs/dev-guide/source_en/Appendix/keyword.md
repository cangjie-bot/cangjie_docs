# Keywords

Keywords are special strings that cannot be used as identifiers. The keywords in the Cangjie language are listed in the following table:

| Keyword       | Keyword      | Keyword     |
|--------------|------------|-----------|
| as           | abstract   | break     |
| Bool         | case       | catch     |
| class        | const      | continue  |
| Rune         | do         | else      |
| enum         | extend     | for       |
| func         | false      | finally   |
| foreign      | Float16    | Float32   |
| Float64      | if         | in        |
| is           | init       | import    |
| interface    | Int8       | Int16     |
| Int32        | Int64      | IntNative |
| let          | mut        | main      |
| macro        | match      | Nothing   |
| open         | operator   | override  |
| prop         | public     | package   |
| private      | protected  | quote     |
| redef        | return     | spawn     |
| super        | static     | struct    |
| synchronized | try        | this      |
| true         | type       | throw     |
| This         | unsafe     | Unit      |
| UInt8        | UInt16     | UInt32    |
| UInt64       | UIntNative | var       |
| VArray       | where      | while     |

After local mode is enabled with the compilation options `--experimental --enable-local-mode`, `exclave` and `demode` are also keywords, used for modal types. For details, see [Modal Types](../modal_type/modal_overview.md). When local mode is not enabled, they are ordinary identifiers. To use a keyword as an identifier, use a raw identifier. For the syntax, see [Identifiers](../basic_programming_concepts/identifier.md).

`local` is not a keyword. It denotes a mode only when it follows the `@` of a modal type, and can still be used as an ordinary identifier elsewhere.
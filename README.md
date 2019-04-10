## EOSIO.CDT Ricardian Contract Specification ![EOSIO Alpha](https://img.shields.io/badge/EOSIO-Alpha-blue.svg)

**Spec Version**: v0.1.0

### General Information
In conjunction with [Ricardian Template Toolkit](https://github.com/EOSIO/ricardian-template-toolkit/), this specification is a new addition to the [EOSIO.CDT](https://github.com/EOSIO/eosio.cdt) suite of tooling.

The Ricardian Contract Specification gives a set of requirements that a Ricardian contract should follow in order to be considered valid. Invalid contracts may still work with non-validating user agents, but will not work with user agents which are built with the [Ricardian Template Toolkit](https://github.com/EOSIO/ricardian-template-toolkit/).

By conforming to a common specification, contracts can be validated and presented in a common way. For instance, Resources must contain a SHA-256 hash value which will allow a validating user agent to check the resource contents and ensure that the resource at the URL has not been altered. Since resources can be used to represent the user or company that is proposing the contract, it is important that the resource URL for the contract is correct and has not been altered since the contract was published.

Ricardian contracts should be written in the English language. The contract itself consists of a set of metadata values supplied in JSON format or YAML format, followed by the body of the contract, written using a subset of [CommonMark](https://commonmark.org/) with [Handlebars](https://handlebarsjs.com/)-based variable substitution.

### Metadata Fields

* `spec-version` -- *Required*
  * Specifies the version of the Ricardian Contract Specification that the contract follows
  * Value is a string in the form of `M.m.p`, where each version element is an integer representing the MAJOR, MINOR, and PATCH level of the specification being followed
  * May also be specified as `M.m`, omitting the PATCH level (*see below* for explanation of compatibility)
  * Implementations which process contracts *MUST* follow the version interpretation given below
* `title` -- *Required*
  * Provides a user-friendly alternative to the action name, representing the title of the action
  * Must not contain variables
  * Maximum length of 50 characters (but keeping this as short as possible is recommended, as some user agents may truncate longer titles)
* `summary` -- *Required*
  * Provides a brief, user-friendly description of the intent of the action
  * Maximum length of 116 characters
  * May contain variables, but content with expanded variables must be within maximum length
* `icon` -- *Required*
  * Provides a user-friendly graphical representation of the intent of the action
  * Is required to be a 256x256 pixel PNG image
  * Must be a valid Resource (*see below*)
  * Must include the SHA-256 hash. Contracts containing an icon without hashes will be rejected.
  * This icon may be displayed alongside the Ricardian contract
* `resources`
  * Contains a map of resources (*see below*) that are used in the Ricardian contract.

### Other Metadata Fields

User-defined metadata fields are permissible with one exception. The `eosio` field name is reserved for future use. Contracts containing that field will be rejected.

User-defined metadata fields may contain variables, which will be replaced during contract rendering.

### Other Requirements and Considerations

#### Specification Versioning

The specification version follows a semantic versioning (semver) inspired scheme. This determines the compatibility that may be expected between versions. Versions are specified in the form of `MAJOR.MINOR.PATCH` where each element is an integer. Interpretation of these elements is as follows:

**MAJOR**

* Differences of `MAJOR` values imply NO guarantees of compatibility between versions.
* This will indicate a change in the specification that cannot be forward compatible. For example, adding a new REQUIRED field to the metadata.

**MINOR**

* Versions having the same `MAJOR` version but different `MINOR` versions must be forward compatible. Processors of a given `MAJOR.MINOR` version *MUST* be able to successfully process contracts having the same `MAJOR` version and the *SAME OR LOWER* `MINOR` version.
* Processor *ARE NOT* required to be able to handle contracts written to a greater `MINOR` version than designed for.
* This will indicate a change in the specification that is forward compatible. For example, making a previously required field optional.

**PATCH**

* Versions differing only in the `PATCH` version *MUST* be processable by any processor with the same `MAJOR.MINOR` version.
* This will indicate changes in the specification that have no tangible effect on processing. For example, a textual change to the specification intended for human readability, spelling corrections, etc.

#### Resources

Contracts may contain inline resources. **Notice:** to specify resources, the metadata values **must be supplied in JSON format**, not YAML. Please see the [README for v0.0.0](https://github.com/EOSIO/ricardian-spec/blob/v0.0.0/README.md#images) for an example of how to define images in YAML format.

A resource is defined as follows:
```
{
  type: string
  hash: string,
  urls: string[]
}
```
Currently the only supported type is `image`.

Resources that do not have a SHA-256 `hash` specified will be rejected. In addition, validating user agents may check that the SHA-256 hash of the resource data (as pulled from any one of the specified urls) matches the supplied hash, and reject the contract if the hashes do not match.

#### Variables

All variables used within a contract must have a value or the contract will be rejected. Variable values may be supplied in the transaction, in the contract action, or in the Ricardian clause on the ABI. In addition, the Ricardian Template Toolkit will supply variables about the transaction.

The default scope for variables is the current action's `data` object. The `eosio.token::transfer` action, for example, has `from`, `to`, `quantity` and `memo` properties. Those variables may be referenced directly: `I am sending {{quantity}} tokens from {{from}} to {{to}}.`.

The Ricardian Template Toolkit will supply the following variables, which allow the contract to refer to other parts of the transaction:

* `$transaction` &ndash; Provides access to transaction-level data like `expiration` or `delay_sec`. E.g., `...with a delay of {{$transaction.delay_sec}} seconds.`
* `$action` &ndash; Provides access to variables within the current action. E.g., `...using the {{$action.name}} contract on the {{$action.account}} account.`
* `$clauses.clause_id` &ndash; Provides access to data in the `ricardian_clauses` on the ABI. E.g., `Section 2:\n\n{{$clauses.standard_clause}}`
* Items in an array may be accessed by index. E.g., Accessing data in another action &ndash; `{{$transaction.actions.[2].data.from}}`
* `$index` &ndash; A pseudo-field giving you the index of the current item in the array you're within. E.g., `{{$action.$index}}`	
* `$resources` &ndash; Provides access to the content that is retrieved for the matching resource from the metadata fields.

Variables may include a reference to other variables. A maximum number of three variable interpolation passes will be made. If there are still unresolved variables after the third pass, the contract will be considered invalid and an error will be returned by the user agent.

For styling purposes, by default interpolated variables in the output will be wrapped in `<div />` tags, all of which will have the class `variable` and one additional contextual class name:

* Values coming from the default context (the current action's `data` values) will be given the `data` class (e.g., `<div class="variable data">value</div>`)
* Transaction values will be given the `transaction` class (e.g., `<div class="variable transaction">value</div>`)
* Action values will be given the `action` class (e.g., `<div class="variable action">value</div>`)
* Clause values will be given the `clauses` class (e.g., `<div class="variable clauses">value</div>`)

In cases where this default variable wrapping can cause problems (e.g. a variable containing a URL that becomes the value of an HREF attribute) there are additional Handlebars handlers to override the default behavior.

* To prevent an individual variable from being wrapped, use the `nowrap` handler.

  For example, to create a link with unwrapped individual variables:

  `[{{nowrap link.text}}]({{nowrap link.url}})`

* To wrap a larger block, perhaps because it contains a link for an unwrapped variable, use the `{{#wrap}}...{{/wrap}}` pair around the block to highlight.

  For example, to wrap the entire link from above:

  `{{#wrap}}[{{nowrap link.text}}]({{nowrap link.url}}){{/wrap}}`

#### HTML in variables

By default, any `{{variables}}` that contain HTML will have that HTML escaped. This may not be the desired outcome, as the `ricardian_clauses` may have HTML that the author wants rendered rather than escaped.

To accommodate this, `{{{variables}}}` wrapped in triple brackets will have HTML escaping disabled. The rendered HTML will be sanitized to reduce the risk of malicious scripts.

#### Metadata Values Beginning With a Variable or Other Special Characters

Metadata values beginning with special characters, such as a variable bracket (`{`), may not be parsed properly by some engines. Ricardian metadata is YAML, and YAML values beginning with special characters are not valid. To get around this, wrap any metadata values beginning with special characters in single quotes.

### Example Template

The example below shows the metadata values supplied in JSON format. Please see the [README for v0.0.0](https://github.com/EOSIO/ricardian-spec/blob/v0.0.0/README.md#example-template) for an example of the metadata values supplied in YAML format.

```
---
{
  "title": "Create Post",
  "summary": "Create a blog post \"{{title}}\" by {{author}} tagged as \"{{tag}}\"",
  "icon": {
    "type": "image",
    "hash": "00506E08A55BCF269FE67F202BBC08CFF55F9E3C7CD4459ECB90205BF3C3B562",
    "urls": [
      "https://app.com/create-post.png",
      "https://dev.app.com/create-post.png"
    ]
  },
  "resources": {
    "content": {
      "type": "image",
      "hash": "1324FECCDDBB89089089090",
      "urls": [
        "https://app.com/user-1/profile-pic.jpg",
      ]
    }
  }
}
---
I, {{author}}, author of the blog post "{{title}}", certify that I am the original author of the contents of this blog post and have attributed all external sources appropriately.

{{$resources.content}}

{{$clauses.legalese}}
```

### Example ABI

```json
"actions": [
  {
    "name": "createpost",
    "type": "createpost",
    "ricardian_contract": "---\n{\n  \"title\": \"Create Post\",\n  \"summary\": \"Create a blog post \\\"{{title}}\\\" by {{author}} tagged as \\\"{{tag}}\\\"\",\n  \"icon\": {\n    \"type\": \"image\",\n    \"hash\": \"00506E08A55BCF269FE67F202BBC08CFF55F9E3C7CD4459ECB90205BF3C3B562\",\n    \"urls\": [\n      \"https://app.com/create-post.png\",\n      \"https://dev.app.com/create-post.png\"\n    ]\n  },\n  \"resources\": {\n    \"content\": {\n      \"type\": \"image\",\n      \"hash\": \"1324FECCDDBB89089089090\",\n      \"urls\": [\n        \"https://app.com/user-1/profile-pic.jpg\",\n      ]\n    }\n  }\n}\n---\nI, {{author}}, author of the blog post \"{{title}}\", certify that I am the original author of the contents of this blog post and have attributed all external sources appropriately.\n\n{{$resources.content}}\n\n{{$clauses.legalese}}"
  }
],

...

"ricardian_clauses": [
  {
    "id": "legalese",
    "body": "WARRANTY. The invoker of the contract action shall uphold its Obligations under this Contract in a timely and workmanlike manner, using knowledge and recommendations for performing the services which meet generally acceptable standards set forth by EOS.IO Blockchain Block Producers."
  }
]
```

## Contributing

[Contributing Guide](./CONTRIBUTING.md)

[Code of Conduct](./CONTRIBUTING.md#conduct)

## License

[MIT](./LICENSE)

## Important

See LICENSE for copyright and license terms.  Block.one makes its contribution on a voluntary basis as a member of the EOSIO community and is not responsible for ensuring the overall performance of the software or any related applications.  We make no representation, warranty, guarantee or undertaking in respect of the software or any related documentation, whether expressed or implied, including but not limited to the warranties or merchantability, fitness for a particular purpose and noninfringement. In no event shall we be liable for any claim, damages or other liability, whether in an action of contract, tort or otherwise, arising from, out of or in connection with the software or documentation or the use or other dealings in the software or documentation.  Any test results or performance figures are indicative and will not reflect performance under all conditions.  Any reference to any third party or third-party product, service or other resource is not an endorsement or recommendation by Block.one.  We are not responsible, and disclaim any and all responsibility and liability, for your use of or reliance on any of these resources. Third-party resources may be updated, changed or terminated at any time, so the information here may be out of date or inaccurate.

Wallets and related components are complex software that require the highest levels of security.  If incorrectly built or used, they may compromise users’ private keys and digital assets. Wallet applications and related components should undergo thorough security evaluations before being used.  Only experienced developers should work with this software.

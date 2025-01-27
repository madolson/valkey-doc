# Valkey documentation

## Clients

**Important**:  Clients listed here, while fully compatible with Valkey, are not official clients for Valkey.
They are maintained by their original developers.

All clients are listed under language specific sub-folders of [clients](./clients)

The path follows the pattern: ``clients/{language}/github.com/{owner}/{repository}.json``.
The ``{language}`` component of the path is the path-safe representation
of the full language name which is mapped in [languages.json](./languages.json).

Each client's JSON object represents the details displayed on the [clients documentation page](https://valkey.io/docs/clients).

For example [clients/python/github.com/placeholderkv](./clients/python/github.com/placeholderkv/placeholderkv-py.json):

```
{
    "name": "placeholderkv-py",
    "description": "Mature and supported. Currently the way to go for Python.",
    "recommended": true
}
```

## Commands

Valkey commands are described in the `commands.json` file that is auto generated
from the Valkey repo based on the JSON files in the commands folder.
See: https://github.com/valkey-io/valkey/tree/unstable/src/commands
See: https://github.com/valkey-io/valkey/tree/unstable/utils/generate-commands-json.py

For each command there's a Markdown file with a complete, human-readable
description.
We process this Markdown to provide a better experience, so some things to take
into account:

*   Inside text, all commands should be written in all caps, in between
    backticks.
    For example: `INCR`.

*   You can use some magic keywords to name common elements in Valkey.
    For example: `@multi-bulk-reply`.
    These keywords will get expanded and auto-linked to relevant parts of the
    documentation.

Each command will have a description and both RESP2 and RESP3 return values.
Regarding the return values, these are contained in the files:

* `resp2_replies.json`
* `resp3_replies.json`

Each file is a dictionary with a matching set of keys. Each key is an array of strings that,
when processed, produce Markdown content. Here's an example:

```json
{
  ...
  "ACL CAT": [
    "One of the following:",
    "* [Array reply](topics/protocol.md#arrays): an array of [Bulk string reply](topics/protocol.md#bulk-strings) elements representing ACL categories or commands in a given category.",
    "* [Simple error reply](topics/protocol.md#simple-errors): the command returns an error if an invalid category name is given."
  ],
  ...
}
```

**Important**: when adding or editing return values, be sure to edit both files. Use the following
links for the reply type. Note: do not use `@reply-type` specifiers; use only the Markdown link.

**Important**: all links below are under construction.

```md
@simple-string-reply: [Simple string reply](https://valkey.io/topics/protocol#simple-strings)
@simple-error-reply: [Simple error reply](https://valkey.io/topics/protocol#simple-errors)
@integer-reply: [Integer reply](https://valkey.io/topics/protocol#integers)
@bulk-string-reply: [Bulk string reply](https://valkey.io/topics/protocol#bulk-strings)
@array-reply: [Array reply](https://valkey.io/topics/protocol#arrays)
@nil-reply: [Nil reply](https://valkey.io/topics/protocol#bulk-strings)
@null-reply: [Null reply](https://valkey.io/topics/protocol#nulls)
@boolean-reply: [Boolean reply](https://valkey.io/topics/protocol#booleans)
@double-reply: [Double reply](https://valkey.io/topics/protocol#doubles)
@big-number-reply: [Big number reply](https://valkey.io/topics/protocol#big-numbers)
@bulk-error-reply: [Bulk error reply](https://valkey.io/topics/protocol#bulk-errors)
@verbatim-string-reply: [Verbatim string reply](https://valkey.io/topics/protocol#verbatim-strings)
@map-reply: [Map reply](https://valkey.io/topics/protocol#maps)
@set-reply: [Set reply](https://valkey.io/topics/protocol#sets)
@push-reply: [Push reply](https://valkey.io/topics/protocol#pushes)
```

## Styling guidelines

Please use the following formatting rules (aiming for smaller diffs that are easier to review):

* No need for manual lines wrapping at any specific length,
  doing so usually means that adding a word creates a cascade effect and changes other lines.
* Please avoid writing lines that are too long,
  this makes the diff harder to review when only one word is changed.
* Start every sentence on a new line.


## Checking your work

After making changes to the documentation, you can use the [spellchecker-cli package](https://www.npmjs.com/package/spellchecker-cli)
to validate your spelling as well as some minor grammatical errors. You can install the spellchecker locally by running:

```bash
npm install --global spellchecker-cli
```

You can than validate your spelling by running the following

```
spellchecker --no-suggestions -f '**/*.md' -l en-US -q -d wordlist
```

Any exceptions you need for spelling can be added to the `wordlist` file.

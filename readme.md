# permission-sheet-to-shell

Compile access control sheets to POSIX shell scripts.

The program reads JSON-formatted permission sheets from standard input and writes a shell script to standard output. The generated script applies ownership and permission policies using standard system tools.

## Usage

```
cat sheets/* | permission-sheet-to-shell > ensure-permissions
chmod +x ensure-permissions
./ensure-permissions
```

Input is read from standard input. Output is written to standard output. No command-line options are required.

## Input format

Input is interpreted as the contents of an implicit JSON array:

```
[ <stdin contents> ]
```

This allows multiple comma-separated rule fragments to be concatenated via standard input.

Each top-level element must be a rule.

## Rule structure

A rule is an array:

```
[ selector, declaration?, nested_rule... ]
```

- selector: string or array of strings
- declaration: optional object
- nested_rule: zero or more rules

The declaration may be omitted when a rule only groups nested rules.

## Selectors

Selectors define paths and may contain shell glob patterns.

### Forms

Absolute path:

```
"/a/b"
```

Relative path:

```
"c"
```

Glob:

```
"d/*"
```

Selector list:

```
["e", "f/*"]
```

### Composition

Nested relative selectors are joined with their parent:

```
[
  "/a",
  {},
  ["b", {}]
]
```

Produces:

```
/a/b
```

Absolute selectors override the parent context:

```
[
  "/a",
  {},
  ["/x", {}]
]
```

Produces:

```
/x
```

Selector lists are expanded independently. Nested rules are evaluated for each resulting parent selector.

Selectors are inserted directly into shell loops. Globs are expanded when the generated script runs. Selectors must therefore be valid shell words; paths containing whitespace or shell metacharacters other than intended glob syntax are not safely handled.

## Declarations

Declarations are objects describing access control properties.

### Keys

- owner: string specifying the owning user
- group: string specifying the owning group
- permissions: string specifying file and directory permissions
- file_permissions: string overriding permissions for non-directories
- directory_permissions: string overriding permissions for directories
- recursive: boolean enabling recursive application

All properties are optional.

The owner and group may be specified independently. If both are absent, ownership is unchanged.

The permissions property provides default modes for files and directories. The file_permissions and directory_permissions properties override their respective defaults.

If recursive is omitted or false, the declaration applies only to paths matched by its selectors.

## Permissions

Permissions use symbolic notation:

- r: read
- w: write
- x: execute

Three comma-separated parts specify user, group, and other permissions:

```
"rw,r,r"
```

Missing parts are treated as empty.

Permissions are converted into three-digit octal modes for chmod.

### Directory normalization

For directories, read permission automatically implies execute permission.

Examples:

- "r": files 400, directories 500
- "rw": files 600, directories 700
- "rw,r,r": files 644, directories 755
- "rwx,r-x,r-x": files 755, directories 755

Explicit execute permission is preserved for both files and directories.

## Recursive rules

```
{ "recursive": true }
```

Applies the declaration to every existing path selected and its descendants.

Recursive operations use POSIX find. Symbolic links are excluded from recursive ownership and permission changes and are not traversed.

File and directory permissions are applied separately using find predicates. Matching paths are processed using batched command execution.

Recursive traversal is performed by find rather than generated shell functions.

## Nested rules

Rules may contain nested rules:

```
[
  "/a",
  { "recursive": true },
  ["b", { "permissions": "r" }]
]
```

This compiles to two independent operations:

- /a: recursive rule
- /a/b: non-recursive permission rule

Nested rules inherit and extend the parent selector, but do not inherit its declaration properties.

A nested rule may override the parent's recursive effects on matching paths because operations execute in declaration order.

## Example

```
[
  "/root",
  {
    "owner": "root",
    "group": "root",
    "recursive": true,
    "file_permissions": "rw",
    "directory_permissions": "rwx"
  },
  [
    "exe/*",
    { "permissions": "rwx" }
  ]
],
[
  "/srv/http",
  {
    "owner": "nonroot",
    "group": "http",
    "recursive": true
  }
]
```

This applies recursive root ownership and permissions to /root, then grants executable permissions to paths matched by /root/exe/*.

It also recursively assigns user nonroot and group http to /srv/http without changing permissions.

## Output

The generated script:

- uses POSIX shell with error termination enabled
- expands selectors using shell globbing
- applies ownership using chown
- applies permissions using chmod
- uses find for recursive operations
- batches recursive operations where possible
- omits operations for unspecified properties
- does not generate shell helper functions

The generated script requires a POSIX shell and the standard system utilities used by its rules. Recursive rules require POSIX find.

## Properties

- Declarative input
- Deterministic output
- Direct compilation without an intermediate normalized rule representation
- No runtime dependency on Node.js
- No JavaScript dependencies in generated scripts
- Independent execution of generated scripts

The generated script can be stored, reviewed, and executed independently.
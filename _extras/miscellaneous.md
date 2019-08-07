---
layout: page
title: Miscellaneous
permalink: /misc/
---

This is a collection of examples and short notes
about some operations that fall outside the scope of the User Guide
and/or have not yet been implemented in a clean way in the CWL standards.

### Non "`File`" types using `evalFrom`

```yaml
cwlVersion: v1.0  # or v1.1
class: CommandLineTool
requirements:
  InlineJavascriptRequirement: {}

baseCommand: [ echo, "42" ]

inputs: []

stdout: my_number.txt

outputs:
  my_number:
    type: int
    outputBinding:
       glob: my_number.txt
       loadContents: True
       outputEval: $(parselnt(self[0].contents))

  my_number_as_string:
    type: string
    outputBinding:
       glob: my_number.txt
       loadContents: True
       outputEval: $(self[0].contents)
```

### Rename an input file

This example shows how you can change the name of an input file
as part of a tool description.
This could be useful when you are taking files produced from another
step in a workflow and don't want to work with the default names that these
files were given when they were created.

```yaml
requirements:
  InitialWorkDirRequirement:
    listing:
      - entry: $(inputs.src1)
        entryName: newName
      - entry: $(inputs.src2)
        entryName: $(inputs.src1.basename)_custom_extension
```

### Rename an output file

This example shows how you can change the name an output file
from the default name given to it by a tool:

```yaml
cwlVersion: v1.0 # or v1.1
class: CommandLineTool
requirements:
  InlineJavascriptRequirement: {}

baseCommand: []

inputs: []

outputs:
 otu_table:
    type: File
    outputBinding:
      glob: otu_table.txt
      outputEval: ${self[0].basename=inputs.otu_table_name; return self;}
```

### Setting `self`-based input bindings for optional inputs

Currently, `cwltool` can't cope with missing optional inputs if their
input binding makes use of `self`.
Below is an example workaround for this,
pending a more sophisticated fix.

```yaml
#!/usr/bin/env cwl-runner
cwlVersion: v1.0
class: CommandLineTool

requirements: { InlineJavascriptRequirement: {} }

inputs:
  cfg:
    type: File?
    inputBinding:
      prefix: -cfg
      valueFrom: |
        ${ if(self === null) { return null;} else { return self.basename; } }

baseCommand: echo

outputs: []
```

### Model a "one-or-the-other" parameter

Below is an example of how
you can specify different strings to be added to a command line
based on the value given to a Boolean parameter.

```yaml
cwlVersion: v1.0
class: CommandLineTool
requirements:
  InlineJavascriptRequirement: {}
inputs:
  fancy_bool:
     type: boolean
     default: false  # or true
     inputBinding:
        valueFrom: ${if (self) { return "foo";} else { return "bar";}}

baseCommand: echo

outputs: []
```

### Connect a solo value to an input that expects an array of that type

Using [`MultipleInputFeatureRequirement`](https://www.commonwl.org/v1.0/Workflow.html#MultipleInputFeatureRequirement)
along with
[`linkMerge: merge_nested`](https://www.commonwl.org/v1.0/Workflow.html#WorkflowStepInput)

>   merge_nested

> The input must be an array consisting of exactly one entry for each input link.
> If "merge_nested" is specified with a single link, the value from the link must be wrapped in a single-item list.

Which means "create a list with exactly these sources as elements"

Or in other words: if the destination is of type `File[]` (an array of `File`s)
and the source is a single `File` then add `MultipleInputFeatureRequirement` to the Workflow level `requirements`
and add `linkMerge: merge_nested` under the appropriate `in` entry of the destination step.

```yaml
cwlVersion: v1.0
class: Workflow

requirements:
  MultipleInputFeatureRequirement: {}

inputs:
  readme: File

steps:
  first:
    run: tests/checker_wf/cat.cwl
    in:
     cat_in:  # type is File[]
       source: [ readme ]  # but the source is of type File
       linkMerge: merge_nested
    out: [txt]

outputs:
  result:
    type: File
    outputSource: first/txt
```
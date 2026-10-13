---
layout: post
title: "Building tool command lines from database rows"
date: 2025-04-16 15:48:13 +0000
categories: python django configuration
---

Every tool in INSaFLU has a long list of options: thresholds, primers, databases, and modes. Those options can
be changed by a user in the web interface, per project and per sample. So the command line cannot be a
hard-coded string. Instead, each option is a row in the database, and the command line is built from the rows
at run time. This post covers how that works, using the code in `settings/default_parameters.py`.

### Rows, not strings

Each option is a `Parameter` row with a name, a value, a separator, and a scope (software, project, sample).
The default builders create the rows. Here is one, the Snippy minimum coverage option:

```python
parameter = Parameter()
parameter.name = DefaultParameters.SNIPPY_COVERAGE_NAME        # "--mincov"
parameter.parameter = "10"
parameter.type_data = Parameter.PARAMETER_int
parameter.software = software
parameter.union_char = " "
parameter.can_change = True
parameter.range_min = "4"
parameter.range_max = "100"
```

The name is the flag. The value is what follows it. `union_char` is the separator, which is a space here.

### Fetching the rows

`get_parameters` filters the rows by software, project, sample, dataset, and pipeline step. Then
`get_parameters_parsed` passes them to `parse_parameters`:

```python
def get_parameters_parsed(self, software_name, user, type_of_use, project, project_sample,
                          sample, ...):
    parameters = self.get_parameters(software_name, user, type_of_use, project,
                                     project_sample, sample, ...)
    if parameters is None:
        return None
    return self.parse_parameters(parameters, software_name)
```

### Turning rows into text

`parse_parameters` (`default_parameters.py:753-870`) first groups rows by name, keeping the order in which
each name first appears. A name can have several values, each with its own separator:

```python
if parameter.name in dict_out:
    dict_out[parameter.name][1].append(parameter.parameter)
    dict_out[parameter.name][0].append(parameter.union_char)
else:
    vect_order_ouput.append(parameter.name)
    dict_out[parameter.name] = [[parameter.union_char], [parameter.parameter]]
```

Then it writes the name, and each value with its separator. Rows whose value equals `not_set_value` are skipped.
Names wrapped in `[...]` are not written as flags, only their values. That is how a row can carry a value
without a flag name.

Some tools have special cases in the same function (Trimmomatic, DIAMOND, Snippy and Medaka primers, and
`--db`). Those are hard-coded because their syntax does not fit the generic rule.

### Rows that are not flags

Not every row becomes a flag. The clustering model choice is a radio-button row with its own name:

```python
parameter.name = SoftwareNames.SOFTWARE_REMAP_PARAMS_clustering_model_type
parameter.type_data = Parameter.PARAMETER_radio_button
```

Application code reads it directly (`televir_parameters.py:255-278`) and never emits it to a command line.
So one table holds both flags and application settings, and the code has to know which is which.

### What works and what I would change

- Changing an option is a database update. No code change and no redeploy.
- The same rows drive the web form, the command, and the tests.
- The rule that picks flags versus values is implicit. A row named `[--x]` is different from a row named
  `--x`, and that difference is only visible in the parser.
- `union_char` is set by each default builder, so the value for a given option can differ between builders.
  A single table of separators per option type would make that consistent.

For how these commands are then executed and checked, see
[the wrapper post]({% post_url 2022-08-16-one-wrapper-for-every-bioinformatics-tool %}).

---
layout: post
title: "Building tool command lines from database rows"
date: 2025-04-16 15:48:13 +0000
categories: python django configuration
---

Every tool in INSaFLU has a long list of options: thresholds, primers, databases, modes. Users can change them in the
web interface, per project and per sample. That means the command line cannot be a hard-coded string, because the
options differ from run to run. For a while I was not sure how to handle that cleanly. What we ended up with is that
every option is a row in the database, and the command line is assembled from those rows at run time. This post
explains how that works, using `settings/default_parameters.py`.

### An option is a row

Each option is a `Parameter` object with a name, a value, a separator, and a scope: the software, the project, or the
sample. The default builders create the rows. Here is one, the minimum coverage for Snippy:

```python
parameter = Parameter()
parameter.name = DefaultParameters.SNIPPY_COVERAGE_NAME        # "--mincov"
parameter.parameter = "10"
parameter.type_data = Parameter.PARAMETER_int
parameter.union_char = " "
parameter.range_min = "4"
parameter.range_max = "100"
```

The name is the flag, and the value is what follows it. The separator is a space here. The range fields are what the
web form uses to validate input, so the same row drives the form, the command and the tests.

### Fetching the rows

When a tool is about to run, the code asks for its parameters for this software, project and sample. The lookup
filters the rows, and then hands them to the parser:

```python
def get_parameters_parsed(self, software_name, user, type_of_use, project, project_sample, sample, ...):
    parameters = self.get_parameters(software_name, user, type_of_use, project,
                                     project_sample, sample, ...)
    if parameters is None:
        return None
    return self.parse_parameters(parameters, software_name)
```

### Turning rows into text

The parser groups rows by name, keeping the order in which each name first appears. A name can carry several values,
each with its own separator, so the grouping is needed:

```python
if parameter.name in dict_out:
    dict_out[parameter.name][1].append(parameter.parameter)
    dict_out[parameter.name][0].append(parameter.union_char)
else:
    vect_order_ouput.append(parameter.name)
    dict_out[parameter.name] = [[parameter.union_char], [parameter.parameter]]
```

It then writes the name followed by each value and its separator. Rows whose value equals the "not set" value are skipped.
A name wrapped in square brackets is not written as a flag, only its values are. That is how a row can carry a value
without a flag. Some tools need special cases in the same function, such as Trimmomatic, DIAMOND, the primer options and
`--db`, because their syntax does not follow the general rule.

### Rows that are not flags

Not every row becomes a command-line flag. The clustering model choice is a radio-button row with a name of its own,
and the application reads it directly to decide how to traverse the tree. It never reaches a command line. So one table
holds both kinds of setting, and the code has to know which is which. That is convenient, and it is also easy to get
wrong: a row named with brackets and a row named without them look almost the same in the table.

### What I would change

Changing an option is now a database update, with no code change and no redeploy, and I think that part works well.
The implicit rules about flags versus values should be written down in one place. The separator is also set separately
by each default builder, so the same option can end up with different separators depending on who created it. A single
table of separators per option type would make that consistent.

The way these commands are run and checked afterwards is in
[the wrapper post]({% post_url 2022-08-16-one-wrapper-for-every-bioinformatics-tool %}).

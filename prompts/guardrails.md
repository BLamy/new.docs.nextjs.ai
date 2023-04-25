---
description: Load .RAIL files
---

# Guardrails

RAIL files help create structured outputs for prompts. To integrate with [Gaurdrails](https://github.com/ShreyaR/guardrails), create a `.Prompt.rail` file in the `./src/ai/prompts` directory.

These files are built during build time and are accessible as environment variables named after the prompt's filename.

### Creating a Rail Prompt

If the prompt was named `BankRun`\
The file name should be `./src/ai/prompts/BankRun.Prompt.rail`\
And the environment variable will be `process.env.BankRunPrompt`.

## Rail file format

{% code title="./src/ai/prompts/bank_run.Prompt.rail" overflow="wrap" lineNumbers="true" %}
```xml
<rail version="0.1">

<output>
    <object name="bank_run" format="length: 2">
        <string
            name="explanation"
            description="A paragraph about what a bank run is."
            format="length: 200 240"
            on-fail-length="noop"
        />
        <url
            name="follow_up_url"
            description="A web URL where I can read more about bank runs."
            required="true"
            format="valid-url"
            on-fail-valid-url="filter"
        />
    </object>
</output>

<prompt>
Explain what a bank run is in a tweet.

@xml_prefix_prompt

{output_schema}

@json_suffix_prompt_v2_wo_none
</prompt>
</rail>
```
{% endcode %}

For details on the RAIL file format, refer to the [Guardrails Docs](https://shreyar.github.io/guardrails/)

# Zod Validation

## Zod Validation

Zod is a typescript library that allows you to define a schema for your types. The difference between zod and typescript is that zod allows you to validate data at runtime instead of just providing static analysis a build time. Zod will also allow you to make assertions that are difficult to make in typescript such as number max and mins.

{% code title="./src/ai/prompts/NumberGenerator.Prompt.ts" overflow="wrap" lineNumbers="true" %}
```typescript
import '@/ai/prompts/preambles/basic.turbo.Prompt.txt';
import '@/ai/prompts/examples/NumberGenerator.Examples.json';

import { z } from 'zod';
const inputSchema = z.object({
  min: z.number().min(1).max(100),
  max: z.number().min(1).max(100),
});
const outputSchema = z.object({
  result: z.number().min(1).max(100),
});
export type Prompt = "Can you tell me a number between {{min}} and {{max}}?"
export type Input = z.infer<typeof inputSchema>
export type Output = z.infer<typeof outputSchema>
export type Errors = "max must be greater than min" | "json parse error" | "zod validation error" 
```
{% endcode %}

This will allow you to double check the inputs and outputs at runtime.

{% code title="./src/app/api/NumberGenerator.ts" overflow="wrap" lineNumbers="true" %}
```typescript
import { inputSchema, outputSchema } from '@/ai/prompts/NumberGenerator.Prompt';

export const config = {
  runtime: 'edge'
}

export default async function handler(req: Request) {
  const input = inputSchema.parse(req.body);

  const response = await openai.createChatCompletion({ 
    model: "gpt-4",
    messages: [
      {
      role: "system",
      content: process.env.NumberGeneratorPrompt,
      },
      {
        role: "user",
        content: JSON.stringify(input),
      },
    ]
  });

  const output = outputSchema.parse(JSON.parse(response.data.choices[0].text));

  return new Response(output);
}
```
{% endcode %}

If either the input or output is invalid, the request will fail with a 400 error.

{% hint style="warning" %}
If either the input or output is invalid, the request will fail with a 400 error.
{% endhint %}

### Reflection using Zod Errors

Defining an output using Zod doesn't always work perfectly on 3.5-Turbo but if you safeParse the output and then feed the error back in so GPT can reflect on its mistake it will often come up with the right solution when given a 2nd chance.&#x20;

![](<../../.gitbook/assets/image (3).png>)

#### Example

{% code title="./src/ai/prompts/JokeGenerator.Prompt.ts" overflow="wrap" lineNumbers="true" %}
```typescript
import '@/ai/prompts/preambles/basic.turbo.Prompt.txt';
import '@/ai/prompts/examples/JokeGenerator.Examples.json';

import { z } from 'zod';

const jokeTypeSchema = z.union([
    z.literal("funny"),
    z.literal("dumb"),
    z.literal("dad"),
]);

export const inputSchema = z.object({
  count: z.number().min(1).max(10),
  jokeType: jokeTypeSchema,
});

export const outputSchema = z.array(
  z.object({
      setup: z.string(),
      punchline: z.string(),
      explanation: z.custom<`This is a ${z.infer<typeof jokeTypeSchema>} joke because ${string}`>((val) => {
        return /This is a (funny|dad|dumb) joke because (.*)/.test(val as string);
      }),
    })
)

export type Prompt = "Can you tell {{count}} {{jokeType}} jokes?"
export type Input = z.infer<typeof inputSchema>
export type Output = z.infer<typeof outputSchema>
```
{% endcode %}

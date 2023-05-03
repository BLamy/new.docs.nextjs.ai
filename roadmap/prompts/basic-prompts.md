# Basic Prompts

Basic prompts are ideal for simple prompts that don't require specific input variables or structured outputs.

### Creating a Basic Prompt

If the prompt was a `Poem Generator` prompt, the prompt file would look like this:

{% code title="@/prompts/poemGenerator.Prompt.txt" overflow="wrap" lineNumbers="true" %}
```
Write a Poem
```
{% endcode %}

And the environment variable will be `process.env.PoemGeneratorPrompt`.

{% code title="src/app/poem/page.tsx" overflow="wrap" lineNumbers="true" %}
```tsx
export default async function PoemPage() {
  const response = await openai.createChatCompletion({ 
    model: "gpt-4",
    messages: [
      {
        role: "user",
        content: process.env.PoemGeneratorPrompt as string,
      }
    ]
  });

  return (
    <div>
      <h1>Poem</h1>
      <p>{response.data.choices[0].text}</p>
    </div>
  );
}
```
{% endcode %}

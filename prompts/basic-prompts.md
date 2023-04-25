# Basic Prompts

Basic prompts are ideal for simple prompts that don't require specific input variables or structured outputs.

### Creating a Basic Prompt

If the prompt was a `Poem Generator` prompt, the prompt file would look like this:

```
Write a Poem
```

And the environment variable will be `process.env.PoemGeneratorPrompt`.

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

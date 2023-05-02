---
description: An Agile Process Framework for the AI-Enabled Age.
---

# Agent Driven Development

In this page, we discuss how to set up your CI/CD pipeline to take full advantage of the power of AI to write code.&#x20;

* Github Issue Templates provide structure to issues and add labels
* When issues are created a CI pipeline which contains a system prompt
  * That pipeline calls out to GPT4 to generate code
  * Opens a pull request whith the generated code
* PR Previews allow you to view changes
  * Visual testing tools like chromatic create visual diffs of changes
* PR comment to revise pull request
  * Revise changes&#x20;
  * Update test&#x20;
  * Update storybooks



### Github Issues -> Github Actions - GPT Code Generation

You can take advantage of [github issue templates](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository#creating-issue-forms) to create forms that trigger code generation.

{% tabs %}
{% tab title="Screenshot" %}
<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>
{% endtab %}

{% tab title="Github Issue template" %}


{% code title="" lineNumbers="true" %}
```yaml
name: Create Atom Component
description: Fill out this form to create a new component
title: "[Atom]: "
labels: ["create-atom"]
body:
- type: input
  id: name
  attributes:
    label: Component Name
    description: Enter the name of the compoent you want to create
    placeholder: ex. SearchBox
  validations:
    required: true
- type: input
  id: desc
  attributes:
    label: Component Description
    description: Enter the description of the component you are trying to make
    placeholder: ex. an input box with large font and a search icon on the left hand side rounded corners 
  validations:
    required: true
```
{% endcode %}
{% endtab %}

{% tab title="Github Action" %}


{% code title="" overflow="wrap" lineNumbers="true" %}
```yaml
name: Create Atom Component
on:
  issues:
    types: [opened]

jobs:
  issue_handler:
    if: contains(github.event.issue.labels.*.name, 'create-atom')
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2

      - name: Print issue information
        run: |
          echo "Issue title: ${{ github.event.issue.title }}"
          echo "Issue body: ${{ github.event.issue.body }}"
          echo "Issue author: ${{ github.event.issue.user.login }}"
      
      - name: Call API and update file
        id: api_call
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          ESCAPED_ISSUE_BODY=$(echo "${{ github.event.issue.body }}" | awk '{printf "%s\\n", $0}')
          RESPONSE=$(curl "https://api.openai.com/v1/chat/completions" \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${{ env.OPENAI_API_KEY }}" \
            -d '{
              "model": "gpt-3.5-turbo",
              "messages": [{"role":"system","content":"You are a react component generator I will feed you a markdown file that contains a component name and description your job is to create a nextjs component using typescript and tailwind do not add any additional libraries or dependencies. Remember if you import useState, useEffect, or useRef you must prefix the file with \"use client\". Your response should only have 1 tsx code block which is the implementation of the component. Do not wrap the code in anything."},{"role": "user", "content": "'"$ESCAPED_ISSUE_BODY"'"}]
            }' \
            --fail)
          echo "::set-output name=response::$RESPONSE"


      - name: Create and switch to new branch
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git checkout -b issue-${{ github.event.issue.number }}-update

      - name: Update file with API response
        run: |
          echo "${{ steps.api_call.outputs.response }}" > src/components/atoms/Search.tsx
          git add .
          git commit -m "Update file with API response"

      - name: Push changes to the new branch
        run: git push origin issue-${{ github.event.issue.number }}-update

      - name: Create pull request
        uses: peter-evans/create-pull-request@v3
        with:
          token: ${{ secrets.GH_API_KEY }}
          title: "Update file with API response for issue #${{ github.event.issue.number }}"
          body: "This PR updates the file with the API response for issue #${{ github.event.issue.number }}."
          branch: issue-${{ github.event.issue.number }}-update

```
{% endcode %}
{% endtab %}
{% endtabs %}

### Use a schema to validate the generated code

When generating code, it is best practice to use a schema to validate the generated code.

For instance, If I were having it generate typescript I may run tsc to validate it's output.

If you were generating a set of rules for an access control system, you could use a zod schema to validate the rules. If GPT ever outputs something that doesn't fit this format you can feed the error back into GPT for reflection.

```typescript
import { z } from 'zod';

export const rulesSchema = z.object({
    roles: z.array(z.string()),
    actions: z.array(z.string()),
    rules: z.array(z.object({
        role: z.string(),
        action: z.string(),
        decision: z.boolean(),
    })),
});
```



### Use automated test to validate generated code

Running a test suite can also be used to create guardrails for the code GPT is generating. In my experience test are best run on PR creation and can be updated from a PR comment. Test should not be updated on the first pass though. Users should have to explicitly ask to have test fixed and the context surrounding the test fix.&#x20;

### Pre-commit hooks to format code

husky pre-commit hooks can be added to format code using formatters like prettier. This will help keep diffs to a minimum.

### Reviewing generated code in a PR

When a user creates an issue using the above template, a github action will run the code generation and submit a PR with the generated code to the repository. This will trigger a new preview build to be created in the PR and you can play around with what the system would be like if those rules were in place. If everything seems fine with the PR demo, you can merge the PR and the new rules will be shipped out to production.&#x20;

<figure><img src="../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

### Ephemeral Environments for PRs

PRs should be staged using something like Vercel/Netlify PR preview and a visual diff regression testing tool like chromatic.&#x20;

<figure><img src="../.gitbook/assets/chromatic-visual-change-approval.gif" alt=""><figcaption><p>Review Changes with Chromatic</p></figcaption></figure>

### Comments in Ephemeral Environments help scope context on issues

{% embed url="https://assets.vercel.com/video/upload/vc_auto/contentful/video/e5382hct74si/1zUv3vaXZhmikxuHyZsDbr/2de0c8108f9ef9316ca01a98cea61c57/Vercel-PreviewCommentsLaunch-Video__2_.mp4" %}

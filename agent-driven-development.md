---
description: An Agile Process Framework for the AI-Enabled Age.
---

# Agent Driven Development

Welcome to Agent Driven Development (ADD), a new agile framework that harnesses the power of AI to revolutionize the software development process. In this post, we'll explore how to set up a CI/CD pipeline to leverage AI to write code, generate storybooks, and streamline your development process.

#### The ADD Process

1. **Structured Github Issues**: ADD uses [GitHub issue templates](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository#creating-issue-forms) to bring structure to your issues and automatically apply labels.
2. **CI Pipeline with AI Integration**: When issues are created, a CI pipeline is triggered, which calls GPT-4 to generate code, create storybooks, and open a pull request with the generated code.
3. **PR Previews**: Review changes in your codebase visually through PR previews.
4. **Visual Testing**: Utilize visual testing tools like Chromatic to create visual diffs of changes.
5. **Comments to Revise Changes**: Update tests, storybooks, and revise changes directly by commenting

## Key Components of Agent Driven Development

### Github Issues Template

To initiate the process, we'll leverage [GitHub issue templates](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository#creating-issue-forms) to create forms that trigger code generation.

{% tabs %}
{% tab title="Screenshot" %}
<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>
{% endtab %}

{% tab title="Code" %}
{% code title=".github/ISSUE_TEMPLATES/create_atom.yml" overflow="wrap" lineNumbers="true" %}
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
{% endtabs %}

### Github Action

You can take advantage of [github issue templates](https://docs.github.com/en/communities/using-templates-to-encourage-useful-issues-and-pull-requests/configuring-issue-templates-for-your-repository#creating-issue-forms) to create forms that trigger code generation.

{% tabs %}
{% tab title="Screenshot" %}
<figure><img src=".gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>
{% endtab %}

{% tab title="Github Action" %}


{% code title=".github/workflows/create_atom.yml" overflow="wrap" lineNumbers="true" %}
````yaml
name: Create Atom Component
on:
  issues:
    types: [opened]

jobs:
  issue_handler:
    if: contains(github.event.issue.labels.*.name, 'create-atom')
    runs-on: ubuntu-latest
    steps:
      - name: Print issue information
        id: setup
        run: |
          ORG_NAME=$(echo "${GITHUB_REPOSITORY}" | cut -d'/' -f1)
          echo "::set-output name=ORG_NAME::$ORG_NAME"
          REPO_NAME=$(echo "${GITHUB_REPOSITORY}" | cut -d'/' -f2)
          echo "::set-output name=REPO_NAME::$REPO_NAME"
          echo "Creating Atom Component for: ${ORG_NAME}/${REPO_NAME}"

          SYSTEM_PROMPT="You are a react component generator I will feed you a markdown file that contains a component name and description your job is to create a nextjs component using typescript and tailwind. Please include a default export & export the Prop as a typescript type. Do not add any additional libraries or dependencies. Your response should only have 1 tsx code block which is the implementation of the component."
          SYSTEM_MESSAGE='{"role":"system","content":"'"$SYSTEM_PROMPT"'"}'
          echo "SYSTEM: $SYSTEM_PROMPT"
          echo "::set-output name=SYSTEM_MESSAGE::$SYSTEM_MESSAGE"

          ESCAPED_ISSUE_BODY=$(echo "${{ github.event.issue.body }}" | awk '{printf "%s\\n", $0}')
          USER_MESSAGE='{"role":"user","content":"'"$ESCAPED_ISSUE_BODY"'"}'
          echo "USER: $ESCAPED_ISSUE_BODY"
          echo "::set-output name=USER_MESSAGE::$USER_MESSAGE"

      - name: Create Component (GPT 3.5 Turbo)
        id: create_component_api_call
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          RESPONSE=$(curl "https://api.openai.com/v1/chat/completions" \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${{ env.OPENAI_API_KEY }}" \
            -d '{
              "model": "gpt-3.5-turbo",
              "messages": [${{ steps.setup.outputs.SYSTEM_MESSAGE }}, ${{ steps.setup.outputs.USER_MESSAGE }}]
            }' \
            --fail)
          RESPONSE_BODY=$(echo "$RESPONSE" | jq -r '.choices[0].message.content')
          CODE_BLOCK=$(echo -e "$RESPONSE_BODY" | sed -n '/^```[a-zA-Z]*$/,/^```$/p' | sed '/^```[a-zA-Z]*\|^```$/d')
          ESCAPED_CODE_BLOCK=$(echo -e "$CODE_BLOCK" | sed 's/"/\\"/g; s/`/\\`/g; s/\$/\\$/g' | awk '{printf "%s\\n", $0}')
          echo "::set-output name=response::$ESCAPED_CODE_BLOCK"
          echo "ASSISTANT: $ESCAPED_CODE_BLOCK"
          COMPONENT_NAME=$(echo -e "$RESPONSE_BODY" | sed -n 's/.*export default \([a-zA-Z_$][0-9a-zA-Z_$]*\).*/\1/p')
          echo "$COMPONENT_NAME"
          echo "::set-output name=COMPONENT_NAME::$COMPONENT_NAME"

      - name: Create Storybook Follow up Prompt
        id: followup    
        run: |
          echo "ASSISTANT: ${{ steps.create_component_api_call.outputs.response }}"
          ESCAPED_RESPONSE=$(echo "${{ steps.create_component_api_call.outputs.response }}" | sed 's/`/\\\\`/g' | sed "s/'/\\'/g" | sed 's/"/\\"/g')
          ASSISTANT_MESSAGE='{"role":"assistant","content":"'"$ESCAPED_RESPONSE"'"}'
          echo "::set-output name=ASSISTANT_MESSAGE::$ASSISTANT_MESSAGE"
          STORYBOOK_FOLLOW_UP_PROMPT="Can you create a storybook for the above component using import ${{ steps.create_component_api_call.outputs.COMPONENT_NAME }}, { ${{ steps.create_component_api_call.outputs.COMPONENT_NAME }}Props } from '../${{ steps.create_component_api_call.outputs.COMPONENT_NAME }}'"
          STORYBOOK_FOLLOW_UP_MESSAGE='{ "role": "user", "content":"'"$STORYBOOK_FOLLOW_UP_PROMPT"'"}'
          echo "::set-output name=STORYBOOK_FOLLOW_UP_MESSAGE::$STORYBOOK_FOLLOW_UP_MESSAGE"

      - name: Create Storybook (GPT 3.5 Turbo)
        id: create_storybook_api_call
        env:
          OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
        run: |
          RESPONSE=$(curl "https://api.openai.com/v1/chat/completions" \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer ${{ env.OPENAI_API_KEY }}" \
            -d '{
              "model": "gpt-3.5-turbo",
              "messages": [${{ steps.setup.outputs.SYSTEM_MESSAGE }}, ${{ steps.setup.outputs.USER_MESSAGE }},${{ steps.followup.outputs.ASSISTANT_MESSAGE }},${{ steps.followup.outputs.STORYBOOK_FOLLOW_UP_MESSAGE }}]
            }' \
            --fail)
          RESPONSE_BODY=$(echo "$RESPONSE" | jq -r '.choices[0].message.content')
          echo "$RESPONSE_BODY"
          echo "----------------------"
          CODE_BLOCK=$(echo -e "$RESPONSE_BODY" | sed -n '/^```[a-zA-Z]*$/,/^```$/p' | sed '/^```[a-zA-Z]*\|^```$/d')
          echo "$CODE_BLOCK"
          echo "----------------------"
          ESCAPED_CODE_BLOCK=$(echo -e "$CODE_BLOCK" | sed 's/"/\\"/g; s/`/\\`/g; s/\$/\\$/g' | awk '{printf "%s\\n", $0}')
          echo "$ESCAPED_CODE_BLOCK"
          echo "::set-output name=response::$ESCAPED_CODE_BLOCK"
          COMPONENT_NAME=$(echo -e "$RESPONSE_BODY" | sed -n 's/.*export default \([a-zA-Z_$][0-9a-zA-Z_$]*\).*/\1/p')
          echo "::set-output name=COMPONENT_NAME::$COMPONENT_NAME"

      # TODO Split Update file in git up into multiple steps
      - name: Update file in git
        env:
          GH_API_KEY: ${{ secrets.GH_API_KEY }}
        run: |
          git clone https://${{ env.GH_API_KEY }}@github.com/${{ steps.setup.outputs.ORG_NAME }}/${{ steps.setup.outputs.REPO_NAME }}.git
          cd nextjs-ai-starter
          git config --global user.email "brettlamy@gmail.com"
          git config --global user.name "Brett Lamy"
          git checkout -b issue-${{ github.event.issue.number }}-update
          echo -e "${{ steps.create_component_api_call.outputs.response }}" > src/components/atoms/${{ steps.create_component_api_call.outputs.COMPONENT_NAME }}.tsx
          echo -e "${{ steps.create_storybook_api_call.outputs.response }}" > src/components/atoms/__tests__/${{ steps.create_component_api_call.outputs.COMPONENT_NAME }}.stories.tsx
          npm run prettier
          git add .
          git commit -m "${{ github.event.issue.title }} - closes #${{ github.event.issue.number }}"
          git push origin issue-${{ github.event.issue.number }}-update

      - name: Create pull request
        env:
          GH_API_KEY: ${{ secrets.GH_API_KEY }}
        run: |
          curl https://api.github.com/repos/${{ steps.setup.outputs.ORG_NAME }}/${{ steps.setup.outputs.REPO_NAME }}/pulls -H "Authorization: token ${{ env.GH_API_KEY }}" -H "Accept: application/vnd.github+json" -X POST -d '{"title":"${{ github.event.issue.title }}", "body":"${{ steps.setup.outputs.ESCAPED_ISSUE_BODY }}", "head":"issue-${{ github.event.issue.number }}-update", "base":"main"}'
````
{% endcode %}
{% endtab %}
{% endtabs %}

### Use a schema to validate the generated code

To ensure the generated code meets your standards, it's a good practice to use a schema to validate it. For example, if you're generating TypeScript code, you might use the TypeScript compiler to validate the output. If the generated code doesn't meet the schema, you can feed that information back into GPT for reflection and improvement.

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

When a user creates an issue using the Github Issue Template, a Github Action generates code and creates a pull request. This pull request can be previewed, allowing you to play around with what the system would be like if those rules were in place. If everything looks good, you can merge the pull request, and the new code will be shipped out to production.

<figure><img src=".gitbook/assets/image (1) (1) (1).png" alt=""><figcaption><p>AI Generated Pull Request</p></figcaption></figure>

### Ephemeral Environments for PRs

PRs should be staged using something like Vercel/Netlify PR preview. This allows for previewing changes in an ephemeral environment before merging to production.

{% embed url="https://assets.vercel.com/video/upload/vc_auto/contentful/video/e5382hct74si/1zUv3vaXZhmikxuHyZsDbr/2de0c8108f9ef9316ca01a98cea61c57/Vercel-PreviewCommentsLaunch-Video__2_.mp4" %}
Vercel PR PReviews
{% endembed %}

### &#x20;Visual Regression testing

Using a tool like Chromatic for visual regression testing helps ensure that changes to the UI don't break anything. Chromatic allows you to review changes and approve them before merging to production.

<figure><img src=".gitbook/assets/chromatic-visual-change-approval.gif" alt=""><figcaption><p>Review Changes with Chromatic</p></figcaption></figure>

###

### Comments in Ephemeral Environments drive WYSIWYG editor.

Finally, using comments in ephemeral environments can help drive a WYSIWYG editor. This allows users to visually edit components and see changes in real-time, making it easier to iterate on designs and quickly prototype new features.

{% embed url="https://assets.vercel.com/video/upload/vc_auto/contentful/video/e5382hct74si/1ubs3OKMuzdEdDoTBePlZV/5f66954ec71f5c338f226feca5a92278/Visual-editing-export-03.mp4" %}
Vercel WYSIWYG Editor
{% endembed %}








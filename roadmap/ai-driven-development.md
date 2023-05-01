---
description: >-
  A Process Framework for the AI-Enabled Age, Collaborate Smarter, Deliver
  Faster.
---

# AI-Driven Development

In this page, we discuss how to set up your CI/CD pipeline to take full advantage of the power of AI to write code.&#x20;

* Github Action to trigger the creation of a PR
* Github Issue template to trigger creation of a PR
* PR Preview
* PR comment to revise pull request
* Update failing test

{% tabs %}
{% tab title="Issue Template" %}
```yaml
name: Update Component
description: Fill out this form to attempt making changes to a component
title: "[Component]: "
labels: ["component-change"]
body:
- type: input
  id: body
  attributes:
    label: Body
    description: Enter your change request
    placeholder: ex. make text bigger
  validations:
    required: true
- type: dropdown
  id: component
  attributes:
    label: Component
    description: Select Component to modify
    options:
    - label
    - button
    - avatar
  validations:
    required: true
```
{% endtab %}

{% tab title="Issue Created Action" %}
```yaml
name: Issue Created Action
on:
  issues:
    types: [opened]

jobs:
  issue_handler:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
      
      - name: Call API and update file
        id: api_call
        env:
          API_TOKEN: ${{ secrets.API_TOKEN }}
        run: |
          RESPONSE=$(curl --request POST \
            --url https://api.example.com/endpoint \
            --header 'authorization: Bearer ${{ env.API_TOKEN }}' \
            --header 'content-type: application/json' \
            --data '{
              "issue_title": "${{ github.event.issue.title }}",
              "issue_body": "${{ github.event.issue.body }}",
              "issue_author": "${{ github.event.issue.user.login }}"
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
          echo "${{ steps.api_call.outputs.response }}" > updated_file.txt
          git add updated_file.txt
          git commit -m "Update file with API response"

      - name: Push changes to the new branch
        run: git push origin issue-${{ github.event.issue.number }}-update

      - name: Create pull request
        uses: peter-evans/create-pull-request@v3
        with:
          token: ${{ secrets.GITHUB_TOKEN }}
          title: "Update file with API response for issue #${{ github.event.issue.number }}"
      body: "This PR updates the file with the API response for issue #${{ github.event.issue.number }}."
      branch: issue-${{ github.event.issue.number }}-update
```
{% endtab %}

{% tab title="PR Comment Action" %}
```yaml
name: PR Comment Triggered Action
on:
  issue_comment:
    types: [created]
  pull_request:
    types: [opened]

jobs:
  pr_comment_handler:
    if: github.event.issue.pull_request
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v2
        with:
          ref: refs/pull/${{ github.event.issue.number }}/head
          fetch-depth: 0

      - name: Make further updates based on user feedback
        env:
          API_TOKEN: ${{ secrets.API_TOKEN }}
        run: |
          # Your script to parse the user's feedback and call the API with the appropriate instructions

      - name: Commit and push the changes
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git commit -am "Update based on user feedback"
          git push
```
{% endtab %}
{% endtabs %}

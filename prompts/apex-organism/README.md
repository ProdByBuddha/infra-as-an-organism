# Digital Apex Organism Prompt Library

This library contains system prompts designed to help practitioners scaffold a "digital apex organism" across multiple implementation languages and runtime environments. Each prompt is written to guide an advanced language model or orchestration agent to deliver production-grade, reliability-first infrastructure blueprints aligned with the organism metaphor from the technical edition.

## Structure

```
prompts/apex-organism/
  README.md
  <environment>/
    <language>.md
```

- **Environment folders** represent primary deployment contexts (cloud-native, edge, hybrid, on-premises, serverless).
- **Language files** provide a specialized system prompt tuned for the tooling ecosystem typical of that language.

## Usage

Select the folder that matches your target runtime environment, then choose the language file that fits your implementation stack. Feed the prompt to your agent or LLM before providing project-specific requirements. The prompt will prime the model to deliver apex-quality scaffolding, covering governance, observability, security, testing, and automation patterns appropriate for that environment.

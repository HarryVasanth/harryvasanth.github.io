---
layout: post
title: "Comprehensive DevOps Guide: CI/CD Pipeline with GitHub Actions"
date: 2024-11-15 14:21:11 +01
categories: devops github
tags: ci-cd pipelines github-actions
---

## Intro

GitHub Actions is a versatile CI/CD platform that allows developers to automate workflows directly in their repositories. This guide provides an in-depth walkthrough of creating advanced CI/CD pipelines for a Node.js application, incorporating reusable workflows, release management, deployment strategies, matrix builds, and more.

---

## Step 1: Create a Workflow File

Start by creating a `.github/workflows/ci-cd.yml` file in your repository:

```yml
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set Up Node.js Environment
        uses: actions/setup-node@v4
        with:
          node-version: "22"

      - name: Install Dependencies
        run: npm install

      - name: Run Tests
        run: npm test

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Deploy Application (Example)
        run: echo "Deploying application..."
```

### Enhancements Added:

- **Pull Request Trigger**: The workflow now triggers on both `push` and `pull_request` events.
- **Environment Context**: Deployment jobs specify an environment (e.g., `production`) for better control.

---

## Step 2: Advanced Features

### **Reusable Workflows**

Reusable workflows allow you to centralize common CI/CD processes for multiple repositories.

#### Example Reusable Workflow (`.github/workflows/reusable.yml`):

```yml
name: Reusable Workflow

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
    secrets:
      API_KEY:
        required: true

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set Up Node.js Environment
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}

      - name: Install Dependencies and Test
        run: |
          npm install
          npm test
```

#### Calling the Reusable Workflow:

```yml
jobs:
  use-reusable-workflow:
    uses: your-org/your-repo/.github/workflows/reusable.yml@main
    with:
      node-version: "22"
    secrets:
      API_KEY: ${{ secrets.API_KEY }}
```

Reusable workflows reduce duplication and ensure consistent CI/CD practices across projects.

---

## Step 3: Matrix Builds for Parallel Testing

Matrix builds allow testing across multiple environments or configurations simultaneously.

```yml
jobs:
  test-matrix:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        node-version: ["18", "20", "22"]
        os: [ubuntu-latest, windows-latest]

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Set Up Node.js Environment
        uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}

      - name: Install Dependencies and Test on ${{ matrix.os }}
        run: |
          npm install
          npm test -- --coverage
```

Matrix builds improve coverage by testing across multiple configurations simultaneously.

---

## Step 4: Release Management Automation

Automate releases with GitHub Actions to streamline versioning and changelogs.

```yml
name: Create Release

on:
  push:
    tags:
      - "v*"

jobs:
  release-job:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Create Release Notes and Publish Release
        uses: Consuma/release-management@v1
        with:
          typeOfRelease: patch # or major/minor/custom based on input.
          githubToken: ${{ secrets.GITHUB_TOKEN }}
```

This workflow triggers when a tag starting with `v` is pushed, automating the release process.

---

## Step 5: Deployment Strategies and Protection Rules

GitHub Actions supports deploying to environments like staging or production with protection rules.

### Example Deployment Workflow with Protection Rules:

```yml
jobs:
  deploy-to-prod:
    runs-on: ubuntu-latest

    environment:
      name: production
      url: https://your-app.com

    steps:
      - name: Deploy Application to Production
        run: echo "Deploying to production..."

    concurrency:
      group: prod-deployments-${{ github.ref }}
      cancel-in-progress: true # Ensures only one deployment at a time.
```

### Features Included Here:

- **Environment Protection Rules** (e.g., manual approvals).
- **Concurrency Management** to prevent conflicts during simultaneous deployments.

---

## Step 6. Scheduled Workflows and Event-Based Triggers

Use cron syntax for scheduled tasks or trigger workflows based on specific GitHub events.

#### Example Scheduled Workflow:

```yml
on:
  schedule:
    - cron: "0 0 * * *" # Runs daily at midnight UTC.
```

#### Event-Based Trigger Example:

```yml
on:
  issue_comment:
    types:
      - created # Triggered when a comment is added to an issue.
```

These triggers provide flexibility for automating maintenance tasks or responding to custom events.

---

## Conclusion

This guide demonstrates how to leverage GitHub Actions for advanced CI/CD pipelines. By incorporating reusable workflows, matrix strategies, release automation, and deployment best practices, you can create scalable and efficient pipelines tailored to your project's needs. Explore GitHub's Marketplace for additional integrations to further enhance your workflows!

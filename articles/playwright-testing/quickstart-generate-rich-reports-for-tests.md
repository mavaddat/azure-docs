---
title: 'Quickstart: Generate rich reports for Playwright tests'
description: 'This quickstart shows how to troubleshoot your test runs using Microsoft Playwright Testing Preview.'
ms.topic: quickstart
ms.date: 09/23/2024
ms.custom: playwright-testing-preview
---

# Quickstart: Troubleshoot tests with Microsoft Playwright Testing Preview

In this quickstart, you learn how to troubleshoot your Playwright tests easily using reports and artifacts published on Microsoft Playwright Testing Preview. Additionally, this guide demonstrates how to utilize the reporting feature, regardless of whether you're running tests on the cloud-hosted browsers provided by the service.

After you complete this quickstart, you'll have a Microsoft Playwright Testing workspace to view test results and artifacts in the service portal.

> [!IMPORTANT]
> Microsoft Playwright Testing is currently in preview. For legal terms that apply to Azure features that are in beta, in preview, or otherwise not yet released into general availability, see the [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/).

## Background

Microsoft Playwright Testing service enables you to:

- Accelerate build pipelines by running tests in parallel using cloud-hosted browsers.
- Simplify troubleshooting by publishing test results and artifacts to the service, making them accessible through the service portal.

These two features of the service can be used independently of each other and each has its own [pricing plan](https://aka.ms/mpt/pricing). This means you can:

- Expedite test runs and streamline troubleshooting by running tests in cloud-hosted browsers and publishing results to the service.
- Run tests only in cloud-hosted browsers to finish test runs faster.
- Publish test results to the service while continuing to run tests locally for efficient troubleshooting.

## Prerequisites

* An Azure account with an active subscription. If you don't have an Azure subscription, create a [free account](https://azure.microsoft.com/free/?WT.mc_id=A261C142F) before you begin.
* Your Azure account needs the [Owner](/azure/role-based-access-control/built-in-roles#owner), [Contributor](/azure/role-based-access-control/built-in-roles#contributor), or one of the [classic administrator roles](/azure/role-based-access-control/rbac-and-directory-admin-roles#classic-subscription-administrator-roles).
* A Playwright project. If you don't have project, create one by using the [Playwright getting started documentation](https://playwright.dev/docs/intro) or use our [Microsoft Playwright Testing sample project](https://github.com/microsoft/playwright-testing-service/tree/main/samples/get-started).
* Azure CLI. If you don't have Azure CLI, see [Install Azure CLI](/cli/azure/install-azure-cli).

## Create a workspace

To get started with publishing test results on Playwright Testing service, first create a Microsoft Playwright Testing workspace in the Playwright portal.

[!INCLUDE [Create workspace in Playwright portal](./includes/include-playwright-portal-create-workspace.md)]

When the workspace creation finishes, you're redirected to the setup guide.

## Install Microsoft Playwright Testing package 

To use the service, install the Microsoft Playwright Testing package. 

```npm
npm init @azure/microsoft-playwright-testing
```

This command generates `playwright.service.config.ts` file which serves to:

- Direct and authenticate your Playwright client to the Microsoft Playwright Testing service.
- Adds a reporter to publish test results and artifacts.

If you already have this file, the prompt asks you to override it. 

To use only reporting feature for the test run, disable cloud-hosted browsers by setting `useCloudHostedBrowsers` as false. 

```typescript
export default defineConfig(
  config,
  getServiceConfig(config, {
    timeout: 30000,
    os: ServiceOS.LINUX,
	useCloudHostedBrowsers: false // Do not use cloud hosted browsers
  }),
  {
    reporter: [['list'], ['@azure/microsoft-playwright-testing/reporter']], // Reporter for Microsoft Playwright Testing service
  }
);
```
Setting the value as `false` ensures that cloud-hosted browsers aren't used to run the tests. The tests run on your local machine but the results and artifacts are published on the service. 

> [!TIP]
> If you wish to accelerate your test run using cloud-hosted browser, you can set `useCloudHostedBrowsers` as true. This will run your tests on the service managed browsers.

## Configure the service region endpoint

In your setup, you have to provide the region-specific service endpoint. The endpoint depends on the Azure region you selected when creating the workspace.

To get the service endpoint URL:

1. In **Add region endpoint in your setup**, copy the region endpoint for your workspace.

    The endpoint URL matches the Azure region that you selected when creating the workspace.

    :::image type="content" source="./media/quickstart-run-end-to-end-tests/playwright-testing-region-endpoint.png" alt-text="Screenshot that shows how to copy the workspace region endpoint in the Playwright Testing portal." lightbox="./media/quickstart-run-end-to-end-tests/playwright-testing-region-endpoint.png":::

## Set up your environment

To set up your environment, you have to configure the `PLAYWRIGHT_SERVICE_URL` environment variable with the value you obtained in the previous steps.

We recommend that you use the `dotenv` module to manage your environment. With `dotenv`, you define your environment variables in the `.env` file.

1. Add the `dotenv` module to your project:

    ```shell
    npm i --save-dev dotenv
    ```

1. Create a `.env` file alongside the `playwright.config.ts` file in your Playwright project:

    ```
    PLAYWRIGHT_SERVICE_URL={MY-REGION-ENDPOINT}
    ```

    Make sure to replace the `{MY-REGION-ENDPOINT}` text placeholder with the value you copied earlier.


## Set up authentication

To publish test results and artifacts to your Microsoft Playwright Testing workspace, you need to authenticate the Playwright client where you're running the tests with the service. The client could be your local dev machine or CI machine. 

The service offers two authentication methods: Microsoft Entra ID and Access Tokens.

Microsoft Entra ID uses your Azure credentials, requiring a sign-in to your Azure account for secure access. Alternatively, you can generate an access token from your Playwright workspace and use it in your setup.

##### Set up authentication using Microsoft Entra ID 

Microsoft Entra ID is the default and recommended authentication for the service. From your local dev machine, you can use [Azure CLI](/cli/azure/install-azure-cli) to sign-in

```CLI
az login
```
> [!NOTE]
> If you're a part of multiple Microsoft Entra tenants, make sure you sign in to the tenant where your workspace belongs. You can get the tenant ID from Azure portal. See [Find your Microsoft Entra Tenant](/azure/azure-portal/get-subscription-tenant-id#find-your-microsoft-entra-tenant). Once you get the ID, sign-in using the command `az login --tenant <TenantID>`

##### Set up authentication using access tokens

You can generate an access token from your Playwright Testing workspace and use it in your setup. However, we strongly recommend Microsoft Entra ID for authentication due to its enhanced security. Access tokens, while convenient, function like long-lived passwords and are more susceptible to being compromised.

1. Authentication using access tokens is disabled by default. To use, [Enable access-token based authentication](./how-to-manage-authentication.md#enable-authentication-using-access-tokens)

2. [Set up authentication using access tokens](./how-to-manage-authentication.md#set-up-authentication-using-access-tokens)

> [!CAUTION]
> We strongly recommend using Microsoft Entra ID for authentication to the service. If you are using access tokens, see [How to Manage Access Tokens](./how-to-manage-access-tokens.md)

## Enable artifacts in Playwright configuration 
In the `playwright.config.ts` file of your project, make sure you're collecting all the required artifacts.
```typescript
  use: {
    trace: 'on-first-retry',
    video:'retain-on-failure',
    screenshot:'on'
  },
```

## Run your tests and publish results on Microsoft Playwright Testing

You've now prepared the configuration for publishing test results and artifacts with Microsoft Playwright Testing. Run tests using the newly created `playwright.service.config.ts` file and publish test results and artifacts to the service.

 ```bash
    npx playwright test --config=playwright.service.config.ts
```
> [!NOTE]
> For the Reporting feature of Microsoft Playwright Testing, you get charged based on the number test results published. If you're a first-time user or [getting started with a free trial](./how-to-try-playwright-testing-free.md), you might start with publishing single test result instead of your full test suite to avoid exhausting your free trial limits.

After the test completes, you can view the test status in the terminal.

```output
Running 6 test using 2 worker
    5 passed, 1 failed (20.2s)
    
Test report: https://playwright.microsoft.com/workspaces/<workspace-id>/runs/<run-id>
```

> [!CAUTION]
> Depending on the size of your test suite, you might incur additional charges for the test results beyond your allotted free test results.

## View test runs and results in the Playwright portal

You can now troubleshoot the failed test cases in the Playwright portal.

[!INCLUDE [View test runs and results in the Playwright portal](./includes/include-playwright-portal-view-test-results.md)]


> [!TIP]
> You can also use Microsoft Playwright Testing service to run tests in parallel using cloud-hosted browsers. Both Reporting and cloud-hosted browsers are independent features and are billed separately. You can use either of these or both. For details, see [How to use service features](./how-to-use-service-features.md)

> [!NOTE]
> The test results and artifacts that you publish are retained on the service for 90 days. After that, they are automatically deleted.  


## Next step

You've successfully created a Microsoft Playwright Testing workspace in the Playwright portal and run your Playwright tests on cloud browsers.

Advance to the next quickstart to set up continuous end-to-end testing by running your Playwright tests in your CI/CD workflow.

> [!div class="nextstepaction"]
> [Set up continuous end-to-end testing in CI/CD](./quickstart-automate-end-to-end-testing.md)

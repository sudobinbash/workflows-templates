
## Capture Document Signatures in Docusign


### <span style="text-decoration:underline;">Overview</span>

Many organizations use Docusign to control agreements – such as Non-Disclosure Agreements (NDAs), Lease Contracts, and Terms of Service (TOS) – that dictate which resources users can access. This template leverages Docusign webhooks to capture when a user signed a document. This information is used by Okta do determine which systems a user can access.

### <span style="text-decoration:underline;">Before you get Started / Prerequisites</span>

Before you get started, you will need:

*   Access to an Okta tenant with Okta Workflows enabled for your org 
*   Active users
*   Access to a Docusign tenant with administrative rights to create a Docusign Connect integration

**Tip:** You can get a docusign tenant for development purposes on developers.docusign.com.


### <span style="text-decoration:underline;">Setup Steps</span>

From the `Capture Document Signatures in Docusign` page, click *Add template*.
A workflow folder named `WebhookDocusign` will be imported.

Create an attribute, group membership rule, and group to track users that signed a contract:

1. In your okta admin dashboard (`https://<tenant>-admin.okta.com/admin/dashboard`), go to *Directories* > *Profile Editor*
2. On the row User (default), click *Profile*
3. Click *Add Attribute*
4. Create an attribute to store the document signature date. For example:
    - Type: String
    - Display Name: NDA signed date
    - Variable Name: ndaDate
    - Description: The date a user signed the NDA
    - User Permission: Read Only
5. Click *Directories* > *Groups*
6. Click *Add Group* and follow the instructions to create a group for users who signed a contract. (i.e. *Users Under NDA*).
7. Click the *Rules* tab
8. Create a rule to add users to the Under NDA Group when the NDA signed date field is filled:
    - Name: Under NDA
    - IF: Use Okta Expression Language (advanced) - `String.len(user.ndaDate) > 0``
    - THEN: Assign to `Under NDA`
9. Activate the Rule.

Configure the workflow:

1. Click *Workflow* > *Workflows Console* to launch the workflow administrative console
2. Click *WebhookDocusign* > *Docusign.ContractSignedWebhook* flow
3. Make sure a connection is selected for the *Read User* and *Update User* cards
4. On Update User card, click the Gear Icon (bottom right corner), then click Choose Field
5. On the input section, select the field you set to store the document signature date (i.e. NDA signed date)
6. If not selected, drag the completedDateTime field from the Docusign Webhook Listener and drop it under the Update User Card on the NDA signed date field
7. Save the flow
8. Click *Home* > *WebhookDocusign*
9. Enable both the *Docusign.ConrtactSignedWebhook* and the *Util.AuthenticateWebhook* flows.
10. On the *Docusign.ConrtactSignedWebhook*, Click the gear icon, then click *API Access*.
11. Select *Expose as Webhook* and record the invoke URL (You will need that to configure Docusign)

Configure Docusign:

1. Access Docusign as Administrator.
2. Click Settings
3. Under Integrations, click Connect
4. Create a encryption key (for authenticating the Webhook message):
    1. Click Connect Keys.
    2. Click Add Secret Key.
    3. Recort the key generated (you will need that to configure your workflow).
    4. Return to the Connect home page.
4. Create a connect webhook:
    1. Click Add Configuration > Custom.
    2. Enter the following data to configure your webhook to Okta workflows:
        - Name: Okta Workflows
        - URL to Publish: Paste the Invoke URL you got while configuring Okta Workflows
        - Require Acknowledgement: check
        - Data Format: REST 2.1
        - Include Data: select Custom Fields, Documents, and Recipients
        - Associated Users: All users
        - Envelope Events: Select only Envelope Signed/Completed
        - Integration and Security Settings: Select Include HMAC Signature and Enable Mutual TLS.
        - Save and enable the connection
5. Get a file name for the main document to sign:
    1. If you already use Docusign templates, click template and select your template.
    2. On the right-hand side, record the name of the document you use to collect the main signature (i.e. NDA_sample.pdf). (you will need that to configure your workflow).

Link Docusign and Workflows:

1. Return to Okta workflows.
2. Under the Webhook Docusign folder, click Tables and then click Docusign Webhook Config
3. Update the HMAC Private Key with the key copied from Docusign 
    *Note:* This private key is used by Okta workflows to validate if a webhook message is sent by Docusign, providing authentication and non-repudiation
4. Update the Document Name with the document name recorded from your template
    *Note:* This attribute ensures Okta is mapping only the document signature you expect. As of now, Docusign Connect does not provide a direct way in its UI to trigger a webhook only when a specific document template was signed. However, you can configure this in Docusign using the [Envelope-level Connect APIs](https://developers.docusign.com/platform/webhooks/connect/build-listener/)

### <span style="text-decoration:underline;">Testing this Flow</span>

1. In Docusign, send a document for a user who already exists in Okta.
2. Sign the document in Docusign.
3. Back on Okta, go to the *Docusign.ContractSignedWebhook* flow and select Flow History. You should see an execution for the user.
4. Open the user profile and UD. Confirm that the document signed attribute is filled and that the user is member of the NDA Signed group.

### <span style="text-decoration:underline;">Limitations & Known Issues</span>

*   This workflow does not support [aggregated messages](https://developers.docusign.com/platform/webhooks/connect/architecture/) from Docusign (when Docusign sends a single webhook call for multiple signature events).
*   Docusign does not guarantee to send a webhook events immediatelly after a document is signed (Docusign works on a queue architecture). This might incur in delays from a document signature to an event sent to Okta.
*   This workflow will always capture the main signer in a document (aka first signer). You can support documents with multiple signers by modifying the Get Signer card in the Docusign.ContractSignedWebhook flow.

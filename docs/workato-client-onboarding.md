# Document Generation Setup

This guide covers the end-to-end process for onboarding a new client into Workato and connecting them to Rubica Chat with document generation.



## Overview

Every client — regardless of whether they are a CRM user — gets their own Workato workspace. The setup has five main stages:

1. Create the customer workspace and set the correct plan
2. Grant API access via the tenant API collection
3. Import the recipe package into the client workspace
4. Set up file storage directories and upload templates
5. Add users and connect the MCP in Rubica Chat



## Stage 1: Create the Customer Workspace

1. Log in to the **Rubica Workato workspace** (not the client's workspace).
2. Go to **Admin Console > Manage Customers**.
3. Click **Add Customer** and enter the firm name.
4. Set the plan to **OEM Ultimate**.

> OEM Ultimate is required for both MCP access and file storage. These are both essential for document generation to work.

5. Save.



## Stage 2: Grant API Access

1. Still in the Rubica workspace, go to **API Platform > Clients**.
2. Click **Add New Client** (or open the existing client if one already exists).
3. Under **Access**, grant access to the **Tenant API collection** — this is the shared portfolio of cross-tenant APIs (merge document, create image, get templates, facts and figures, etc.).
4. Save.
5. Generate a **new API key** specifically for Rubica Chat. Do not reuse the Salesforce API key.
6. Copy and save this key — you will need it in Stage 3.

> The tenant API architecture means all doc gen recipes live in the central Rubica workspace. Client workspaces call those recipes via API, so you don't need to manage 100 separate recipe sets.


## Stage 3: Import the Recipe Package

1. Navigate to the **client's Workato workspace**.
2. Go to **Tools > Recipe Lifecycle Management > Import**.
3. Select **Import Package** and choose the document generation package file.

> Save the package to a shared Google Drive location so it is always accessible for onboarding.

4. When prompted, set the destination **project to the existing CRM project** (not a new one). The MCP must live in the same project as all other tools — a separate project means a separate MCP endpoint.
5. Once imported, go to **Project Settings** and paste in the **API key** generated in Stage 2 under the Tenant API Token field.
6. Test the connection by running a recipe (e.g. Get Slide Presentation). If it returns a result, the connection is working.
7. Start all remaining recipes. You can bulk-start them directly from the project view.



## Stage 4: File Storage and Templates

1. Go to **Tools > File Storage**.
2. Create two directories:
   - `templates`
   - `merged docs`

> Spelling matters. The merge engine looks for these exact folder names.

3. Upload the client's document templates (SOA, ROA, etc.) into the `templates` folder.



## Stage 5: Add Users and Connect in Rubica Chat

### Add Users to the Workspace

1. In the client workspace, go to **Workspace Admin > Access Control > End Users**.
2. Invite each staff member as an **end user**.
3. Add yourself as a **workspace admin** for testing purposes.

> Tip: keep the same login credentials across all Workato workspaces to avoid MFA/authentication conflicts when testing in Rubica Chat.

### Connect the MCP in Rubica Chat

1. Copy the **Workato MCP URL** from the client workspace.
2. Log in to the client's **Rubica Chat** org.
3. Go to **Connectors > Add Connector**.
4. Set the connector name to the **firm name**.
5. Add the firm logo as the icon.
6. Paste in the MCP URL.
7. Check **I trust this application** and click **Create**.
8. Click **Connect** to authenticate.



## Troubleshooting: MCP Authentication Failure

If the connector fails to authenticate in Rubica Chat and the error says "check the logs":

- Verify the Workato MCP URL is correct.
- Confirm the API key in Project Settings is the Rubica Chat key (not the Salesforce key).
- Test with a minimal recipe set (even a single skill recipe) to rule out a recipe-level connection issue.
- Check that other client workspaces are connecting fine to isolate whether the issue is workspace-specific.
- If the problem persists across empty workspaces and fresh setups, escalate to Workato support (contact Ricardo).



## Summary Checklist

- [ ] Customer added in Admin Console, plan set to OEM Ultimate
- [ ] Client added in API Platform with Tenant API collection access
- [ ] Rubica Chat API key generated and copied
- [ ] Recipe package imported into CRM project
- [ ] Tenant API token key added to project settings
- [ ] Connection tested successfully
- [ ] All recipes started
- [ ] `templates` and `merged docs` directories created in file storage
- [ ] Templates uploaded to `templates` folder
- [ ] Staff added as end users
- [ ] Self added as workspace admin
- [ ] MCP connector created and authenticated in Rubica Chat

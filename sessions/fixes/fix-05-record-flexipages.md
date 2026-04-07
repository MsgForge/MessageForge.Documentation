# Fix #5: Record Page FlexiPages Need Correct Template Name

**Priority:** P3 — cosmetic
**Agent:** `salesforce-dev` (CLI only)

---

## Current State

`Messenger_Channel_Record_Page.flexipage-meta.xml` and `Messenger_Chat_Record_Page.flexipage-meta.xml` use `flexipage:defaultRecordHomeTemplate` which may fail to deploy depending on org API version.

## Fix Approach

Retrieve the correct flexipage templates directly from the sandbox org (where they were created via Lightning App Builder), then commit the correct XML.

---

## Step 1: Create Record Pages in Sandbox (if not done)

If record pages don't exist in the sandbox yet:
1. Open **Setup → Lightning App Builder → New**
2. Choose **Record Page** → select `Messenger_Chat__c`
3. Pick a template (e.g., "Header and One Column")
4. Add the `messengerChat` component
5. Save and activate
6. Repeat for `Messenger_Channel__c` with standard field sections

## Step 2: Retrieve from Sandbox

```bash
cd MessageForge.Salesforce

# Retrieve all flexipages from the sandbox
sf project retrieve start --metadata FlexiPage --target-org sandbox

# Check what was retrieved
git diff force-app/main/default/flexipages/
```

## Step 3: Verify and Commit

```bash
# Verify the template names are org-specific and valid
grep '<name>' force-app/main/default/flexipages/Messenger_Chat_Record_Page.flexipage-meta.xml
grep '<name>' force-app/main/default/flexipages/Messenger_Channel_Record_Page.flexipage-meta.xml

# Test deploy back (round-trip)
sf project deploy start --source-dir force-app/main/default/flexipages/ --target-org sandbox --dry-run
```

## Step 4: Verify App Pages Still Work

The App Pages (Messenger Console, Channel Setup) already deploy fine. Confirm they weren't affected:

```bash
sf project deploy start --source-dir force-app/main/default/flexipages/Messenger_Console.flexipage-meta.xml --target-org sandbox --dry-run
sf project deploy start --source-dir force-app/main/default/flexipages/Channel_Setup.flexipage-meta.xml --target-org sandbox --dry-run
```

---

## Execution Prompt

```
@salesforce-dev Retrieve the correct Record Page flexipages from the sandbox org.

The current flexipage XML files for Messenger_Chat_Record_Page and Messenger_Channel_Record_Page use template names that fail to deploy. The fix is to retrieve the actual pages from the sandbox where they were created via Lightning App Builder.

Steps:
1. Run: sf project retrieve start --metadata FlexiPage --target-org sandbox
2. Check git diff on force-app/main/default/flexipages/
3. Verify the template <name> values in the retrieved XML
4. Test round-trip deploy: sf project deploy start --source-dir force-app/main/default/flexipages/ --target-org sandbox --dry-run
5. If record pages don't exist in sandbox yet, note that they need to be created via Lightning App Builder UI first

Working directory: MessageForge.Salesforce/
Do NOT modify any flexipage XML manually — only retrieve from org.
```

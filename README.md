# Remote Move Migration to Microsoft 365 (Exchange Online)

This guide describes how to migrate mailboxes from on-premises Exchange to Exchange Online using a **Remote Move migration** in a hybrid deployment. Mailboxes stay online during the move, and users keep their identity, mail flow and free/busy visibility throughout.

Replace the example values (`contoso.com`, `contoso.mail.onmicrosoft.com`, `admin@contoso.com`) with your own.

---

## 1. When to use this method

Use a Remote Move migration when:

- A full or minimal hybrid deployment is configured (Hybrid Configuration Wizard completed).
- Directory synchronisation runs through Microsoft Entra Connect (formerly Azure AD Connect).
- The on-premises organisation runs a supported Exchange version (2013, 2016 or 2019 are recommended; check Microsoft documentation for the current support matrix).
- Mailboxes are to be moved in batches, with minimal user disruption.

Use a different method (cutover, staged, IMAP) if no hybrid deployment exists.

---

## 2. Prerequisites checklist

| Item | Requirement |
|---|---|
| Hybrid configuration | Hybrid Configuration Wizard (HCW) completed successfully |
| Directory sync | Entra Connect running, users synchronised, no sync errors |
| Certificate | Valid certificate from a public CA on the Exchange servers, covering the Autodiscover and EWS namespaces |
| Network | Inbound HTTPS (TCP 443) from Exchange Online to the on-premises Exchange servers |
| Permissions (on-premises) | Member of **Organization Management** |
| Permissions (cloud) | **Exchange Administrator** (or Global Administrator) |
| Licences | One Exchange Online licence available per migrating user |
| DNS | Autodiscover resolves correctly for the on-premises organisation |
| Client versions | Supported Outlook versions in use |

---

## 3. Step-by-step procedure

### Step 1: Plan and assess

1. List all mailboxes to be moved, including shared, resource and archive mailboxes.
2. Record mailbox sizes and item counts:

   ```powershell
   Get-Mailbox -ResultSize Unlimited | Get-MailboxStatistics |
     Select DisplayName, TotalItemSize, ItemCount |
     Export-Csv C:\Migration\MailboxSizes.csv -NoTypeInformation
   ```

3. Identify mailboxes with large items (over 150 MB is the Exchange Online limit) and clean up where needed.
4. Group users into a **pilot batch** (10 to 20 users) and later production batches.
5. Notify users of the migration window and any required Outlook restart.

### Step 2: Verify the hybrid configuration

On an on-premises Exchange Management Shell:

```powershell
Get-HybridConfiguration
Get-OrganizationRelationship
Get-IntraOrganizationConnector
```

Test OAuth connectivity to Exchange Online:

```powershell
Test-OAuthConnectivity -Service EWS -TargetUri https://outlook.office365.com/ews/exchange.asmx -Mailbox user@contoso.com -Verbose
```

The result should show `Success`. Re-run the HCW if any check fails.

### Step 3: Enable MRS Proxy

The Mailbox Replication Service (MRS) Proxy must be enabled on the EWS virtual directory of the Exchange servers that Exchange Online contacts.

```powershell
Set-WebServicesVirtualDirectory -Identity "EXCH01\EWS (Default Web Site)" -MRSProxyEnabled $true
```

Verify:

```powershell
Get-WebServicesVirtualDirectory | Format-List Server, MRSProxyEnabled
```

Repeat for every Exchange server in the external-facing namespace.

### Step 4: Confirm the user accounts are ready

1. Check that each user is synchronised and visible in the Microsoft 365 admin centre.
2. Set a **usage location** for each user.
3. Assign an **Exchange Online licence**.

   > A mailbox that is moved without a licence is disabled after a 30-day grace period. Assign licences before or immediately after the move.

4. Confirm the on-premises mailbox is healthy:

   ```powershell
   Get-MailboxStatistics user@contoso.com
   Get-MailboxRepairRequest -Mailbox user@contoso.com
   ```

### Step 5: Test connectivity to the migration endpoint

Connect to Exchange Online PowerShell:

```powershell
Connect-ExchangeOnline -UserPrincipalName admin@contoso.com
```

Run the availability test:

```powershell
$cred = Get-Credential
Test-MigrationServerAvailability -ExchangeRemoteMove -Autodiscover `
  -EmailAddress admin@contoso.com -Credentials $cred
```

`Result: Success` confirms that Exchange Online can reach the on-premises servers.

### Step 6: Check or create the migration endpoint

The HCW normally creates an endpoint. Check for it:

```powershell
Get-MigrationEndpoint
```

If none exists, create one:

```powershell
New-MigrationEndpoint -ExchangeRemoteMove -Name "Hybrid Endpoint" `
  -Autodiscover -EmailAddress admin@contoso.com -Credentials $cred
```

### Step 7: Prepare the batch CSV file

Create a CSV file with a single column named `EmailAddress`:

```csv
EmailAddress
user1@contoso.com
user2@contoso.com
user3@contoso.com
```

Save the file, for example, as `C:\Migration\Pilot.csv`.

### Step 8: Create the pilot migration batch

```powershell
$csv = [System.IO.File]::ReadAllBytes("C:\Migration\Pilot.csv")

New-MigrationBatch -Name "Pilot01" `
  -SourceEndpoint "Hybrid Endpoint" `
  -TargetDeliveryDomain "contoso.mail.onmicrosoft.com" `
  -CSVData $csv `
  -BadItemLimit 10 `
  -LargeItemLimit 10 `
  -NotificationEmails admin@contoso.com
```

Notes:

- Omit `-AutoComplete` to keep control over the final cutover of each mailbox (recommended for the pilot).
- Add `-AutoComplete` for later batches if the mailboxes should finish without manual action.
- Use `-CompleteAfter "2026-11-01 22:00"` to schedule completion outside working hours.
- Add `-ArchiveOnly` or `-PrimaryOnly` to move the archive or primary mailbox separately.

### Step 9: Start the batch

```powershell
Start-MigrationBatch -Identity "Pilot01"
```

Alternatively, use the Exchange admin centre: **Migration > Batch > Start**.

### Step 10: Monitor progress

Batch overview:

```powershell
Get-MigrationBatch -Identity "Pilot01" | Format-List Status, TotalCount, SyncedCount, FailedCount
```

Per-user status:

```powershell
Get-MigrationUser -BatchId "Pilot01"
Get-MigrationUserStatistics -Identity user1@contoso.com -IncludeReport | Format-List
```

The batch reaches **Synced** status when the initial copy is complete. Incremental synchronisation then runs every 24 hours (or as scheduled) until completion.

### Step 11: Complete the batch

When the batch shows **Synced** (95% complete):

```powershell
Complete-MigrationBatch -Identity "Pilot01"
```

During completion, MRS performs a final synchronisation, converts the on-premises mailbox into a mail user, and activates the cloud mailbox. This usually takes a few minutes per mailbox.

### Step 12: Validate the pilot

1. Confirm the mailbox location:

   ```powershell
   Get-Mailbox user1@contoso.com | Format-List Database, RecipientTypeDetails
   ```

   The type should be `UserMailbox` in Exchange Online. On-premises, the object should now be `RemoteUserMailbox`.

2. Ask pilot users to confirm that:
   - Outlook reconnects (a restart or profile prompt can occur).
   - Send and receive works internally and externally.
   - Calendar, contacts and delegates are intact.
   - Mobile devices reconnect.
3. Send test mail from an on-premises mailbox to a migrated mailbox and the reverse.
4. Check free/busy lookups between on-premises and cloud users.

### Step 13: Migrate the remaining batches

Repeat Steps 7 to 12 for each production batch. Recommended practice:

- Keep batches to a manageable size (for example 50 to 200 mailboxes).
- Run initial syncs during the week and complete batches during off-hours.
- Avoid starting too many concurrent moves; large concurrency can throttle bandwidth and slow users down.

### Step 14: Post-migration tasks

1. Remove completed batches:

   ```powershell
   Remove-MigrationBatch -Identity "Pilot01"
   ```

2. Confirm that all users hold licences.
3. Move shared and resource mailboxes (these do not require a licence if under 50 GB).
4. Verify that mail flow works as expected (connectors, MX records, SPF, DKIM, DMARC).
5. Keep the hybrid environment until all mailboxes are moved and decommissioning has been agreed.

### Step 15: Decommission (optional, after all mailboxes have moved)

1. Move the MX record to Exchange Online Protection.
2. Point Autodiscover to Exchange Online.
3. Retain at least one hybrid Exchange server if the on-premises directory remains the source of authority (needed for recipient management).
4. Follow Microsoft guidance before removing the last Exchange server.

---

## 4. Rollback (offboarding)

To return a mailbox to on-premises, create an offboarding batch in the Exchange admin centre (**Migration > Add migration batch > Migrate from Exchange Online**) or run:

```powershell
New-MoveRequest -Identity user1@contoso.com -OutBound `
  -RemoteTargetDatabase "DB01" `
  -RemoteHostName "mail.contoso.com" `
  -RemoteCredential $cred `
  -TargetDeliveryDomain "contoso.com"
```

---

## 5. Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| `Test-MigrationServerAvailability` fails | MRS Proxy off, firewall or certificate issue | Recheck Step 3, port 443 and the certificate chain |
| User status **Failed** | Bad or large items over the limit | Review `Get-MigrationUserStatistics -IncludeReport`; raise limits or clean up items |
| Batch stays at **Syncing** | Throttling or high concurrency | Reduce concurrent moves, check bandwidth |
| Mailbox disabled after move | No licence assigned | Assign a licence within 30 days |
| Outlook cannot connect after move | Autodiscover points to wrong location | Verify the Autodiscover record and the hybrid SCP |
| Free/busy not working | Organisation relationship or OAuth broken | Re-run HCW; test with `Test-OAuthConnectivity` |

---

## 6. Useful commands

```powershell
Get-MigrationBatch
Get-MigrationUser -BatchId "<name>"
Get-MoveRequest | Get-MoveRequestStatistics
Stop-MigrationBatch -Identity "<name>"
Set-MigrationBatch -Identity "<name>" -BadItemLimit 50 -AllowUnknownColumnsInCsv $true
```

---

## 7. References

- Microsoft Learn: *Move mailboxes between on-premises and Exchange Online organisations in hybrid deployments*
- Microsoft Learn: *Hybrid Configuration Wizard*
- Microsoft Learn: *Exchange Online limits*

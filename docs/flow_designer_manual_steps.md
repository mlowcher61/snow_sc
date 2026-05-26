# Flow Designer — manual build steps

The Outbound REST Message, Business Rule, and Catalog Item are created by Ansible. The Flow
Designer flow is built manually (it cannot be reliably created via the Table API), then linked to
the catalog item.

## Build the flow
1. **Flow Designer → New → Flow.** Trigger: **Service Catalog** item. Click **Done**.
2. Add **Action → Action**, search **Get Catalog Variables**, select it. Drag the **Requested Item
   Record** from the data panel on the right into the drop zone.
3. Add **Ask for Approval**. Drag the **Requested Item** into the drop zone. Under **Approve when**,
   select **Anyone approves**, then choose your approval **Group**.
4. Add **Update Record** to set the Requested Item **State** to **Work in Progress** after approval.
5. **Activate** the flow.

## Link the flow to the catalog item
1. Open the catalog item created by Ansible (its sys_id is printed by the
   `Configure ServiceNow Catalog` job — see the debug task in `catalog_item.yml`).
2. On the **Process Engine** tab, set the **Flow** to the flow you just activated.
3. Confirm the item is **Active** and **Published** so it is orderable.

## Why this is manual
Flow definitions span many interrelated `sys_hub_*` records and are normally moved between
instances via Update Sets, not the Table API. Keeping it manual keeps the automated artifacts
transparent and version-portable.

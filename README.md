# tap-tplcentral

This is a [Singer](https://singer.io) tap that produces JSON-formatted data
following the [Singer
spec](https://github.com/singer-io/getting-started/blob/master/SPEC.md).

[**3PLCentral**](https://3plcentral.com/about-us/) is a leading cloud-based Warehouse Management System (WMS) used by distributors/wholesalers and their partner customers and suppliers.

**This tap:**
- Pulls raw data from the [3PLCentral REST API](http://api.3plcentral.com/rels/)
- Extracts the following resources:
  - [Customers](http://api.3plcentral.com/rels/customers/customers)
    - [SKU Items](http://api.3plcentral.com/rels/customers/items)
    - [Stock Details](http://api.3plcentral.com/rels/inventory/stockdetails)
  - [Inventory](http://api.3plcentral.com/rels/inventory/inventory)
  - [Orders (with Order Items and Packages)](http://api.3plcentral.com/rels/orders/orders)
- Outputs the schema for each resource
- Incrementally pulls data based on the input state

## Streams
[**customers**](http://api.3plcentral.com/rels/customers/customers)
- Endpoint: https://secure-wms.com/customers
- Primary keys: customer_id
- Foreign keys: None
- Replication strategy: Full table (one record)
  - Filter: customer_id (from config.json)
- Transformations: Fields camelCase to snake_case, De-nest and remove nodes (ReadOnly, embedded, links).
- Children: sku_items, stock_details

[**sku_items**](http://api.3plcentral.com/rels/customers/items)
- Endpoint: https://secure-wms.com/customers/[customer_id]/items
- Primary keys: item_id
- Foreign keys: customer_id (customers)
- Replication strategy: Incremental (query filtered)
  - Sort by: ReadOnly.lastModifiedDate ASC
  - Bookmark query field: ReadOnly.lastModifiedDate
  - Bookmark: last_modified_date (date-time)
- Transformations: Fields camelCase to snake_case, De-nest and remove nodes (ReadOnly, embedded, links).
- Parent: customer

[**stock_details**](http://api.3plcentral.com/rels/inventory/stockdetails)
- Endpoint: https://secure-wms.com/inventory/stockdetails
- Primary keys: receive_item_id
- Foreign keys: customer_id (customers), item_identifier > id (sku_items), supplier_identifier > id (suppliers), location_identifier > id (locations), facility_identifier > id (facilities), receiver_id (receivers)
- Replication strategy: Incremental (query filtered)
  - Filters: customer_id, facility_id (from config.json)
  - Sort by: receivedDate ASC
  - Bookmark query field: receivedDate
  - Bookmark: received_date (date-time)
- Transformations: Fields camelCase to snake_case, De-nest and remove nodes (ReadOnly, embedded, links).
- Parent: customer

[**inventory**](http://api.3plcentral.com/rels/inventory/inventory)
- Endpoint: https://secure-wms.com/inventory
- Primary keys: receive_item_id
- Foreign keys: customer_id (customers), receive_item_id (stock_details), item_identifier > id (sku_items), supplier_identifier > id (suppliers), location_identifier > id (locations), facility_identifier > id (facilities), pallet_identifier > id (pallets), receiver_id (receivers)
- Replication strategy: Incremental (query filtered)
  - Filters: customer_id, facility_id (from config.json)
  - Sort by: receivedDate ASC
  - Bookmark query field: receivedDate
  - Bookmark: received_date (date-time)
- Transformations: Fields camelCase to snake_case, De-nest and remove nodes (ReadOnly, embedded, links).

[**orders w/ order_items and packages**](http://api.3plcentral.com/rels/orders/orders)
- Endpoint: https://secure-wms.com/orders
- Primary keys: order_id (also: order_item_id and package_id in list arrays sub-elements)
- Foreign keys: receive_item_id (inventory, stock_details), customer_identifier > id (customers), item_identifier > id (sku_items), location_identifier > id (locations), facility_identifier > id (facilities), pallet_identifier > id (pallets)
- Replication strategy: Incremental (query filtered)
  - Filters: detail = All, itemDetail = All
  - Sort by: ReadOnly.lastModifiedDate ASC
  - Bookmark query field: ReadOnly.lastModifiedDate
  - Bookmark: last_modified_date (date-time)
- Transformations: Fields camelCase to snake_case, De-nest and remove nodes (ReadOnly, embedded, links).

**Endpoints to be added later**: facilities, locations, pallets, suppliers, receivers (initial development client did not have access to these endpoints)

## Quick Start

1. Install

    Clone this repository, and then install using setup.py. We recommend using a virtualenv:
    ```bash
    > virtualenv -p python3 venv
    > source venv/bin/activate
    > python setup.py install
    OR
    > cd .../tap-tplcentral
    > pip install .
    ```
2. Dependent libraries
    The following dependent libraries were installed.
    ```bash
    > pip install singer-python
    > pip install singer-tools
    > pip install target-stitch
    > pip install target-json
    
    ```
    - [singer-tools](https://github.com/singer-io/singer-tools)
    - [target-stitch](https://github.com/singer-io/target-stitch)
3. Create your tap's config file which should look like the following:

    ```json
    {
        "start_date": "2019-01-01T00:00:00Z",
        "client_id": "OAUTH_CLIENT_ID",
        "client_secret": "OAUTH_CLIENT_SECRET",
        "tpl_key": "WH_SPECIFIC_TPL_KEY",
        "user_login_id": "USER_INTEGER_ID",
        "user_agent": "tap-tplcentral <my.email@domain.com>",
        "base_url": "http://secure-wms.com",
        "customer_id": "CUSTOMER_INTEGER_ID",
        "facility_id": "FACILITY_INTEGER_ID"
    }
    ```

4. Run the Tap in Discovery Mode
    This creates a catalog.json for selecting objects/fields to integrate:
    ```bash
    tap-tplcentral --config config.json --discover > catalog.json
    ```
   See the Singer docs on discovery mode
   [here](https://github.com/singer-io/getting-started/blob/master/docs/DISCOVERY_MODE.md#discovery-mode).

5. Run the Tap in Sync Mode (with catalog) and [write out to state file](https://github.com/singer-io/getting-started/blob/master/docs/RUNNING_AND_DEVELOPING.md#running-a-singer-tap-with-a-singer-target)

    For Sync mode:
    ```bash
    > tap-tplcentral --config tap_config.json --catalog catalog.json > state.json
    > tail -1 state.json > state.json.tmp && mv state.json.tmp state.json
    ```
    To load to json files to verify outputs:
    ```bash
    > tap-tplcentral --config tap_config.json --catalog catalog.json | target-json > state.json
    > tail -1 state.json > state.json.tmp && mv state.json.tmp state.json
    ```
    To pseudo-load to [Stitch Import API](https://github.com/singer-io/target-stitch) with dry run:
    ```bash
    > tap-tplcentral --config tap_config.json --catalog catalog.json | target-stitch --config target_config.json --dry-run > state.json
    > tail -1 state.json > state.json.tmp && mv state.json.tmp state.json
    ```

6. Test the Tap
    
    While developing the tplcentral tap, the following utilities were run in accordance with Singer.io best practices:
    Pylint to improve [code quality](https://github.com/singer-io/getting-started/blob/master/docs/BEST_PRACTICES.md#code-quality):
    ```bash
    > pylint tap_tplcentral -d missing-docstring -d logging-format-interpolation -d too-many-locals -d too-many-arguments
    ```

    To [check the tap](https://github.com/singer-io/singer-tools#singer-check-tap) and verify working:
    ```bash
    > tap-tplcentral --config tap_config.json --catalog catalog.json | singer-check-tap >> state.json
    > tail -1 state.json > state.json.tmp && mv state.json.tmp state.json
    ```

---

Copyright &copy; 2019 Stitch

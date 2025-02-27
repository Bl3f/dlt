# Salesforce

:::info Need help deploying these sources, or figuring out how to run them in your data stack?

[Join our Slack community](https://dlthub.com/community)
or [book a call](https://calendar.app.google/kiLhuMsWKpZUpfho6) with our support engineer Adrian.
:::

[Salesforce](https://www.salesforce.com) is a cloud platform that streamlines business operations and customer relationship
management, encompassing sales, marketing, and customer service.

This Salesforce `dlt` verified source and
[pipeline example](https://github.com/dlt-hub/verified-sources/blob/master/sources/salesforce_pipeline.py)
loads data using “Salesforce API” to the destination of your choice.

The resources that this verified source supports are:

| Name           | Mode    | Description                                                                                       |
|----------------|---------|---------------------------------------------------------------------------------------------------|
| User           | replace | refers to an individual who has access to a Salesforce org or instance                            |
| UserRole       | replace | a standard object that represents a role within the organization's hierarchy                      |
| Lead           | replace | prospective customer/individual/org. that has shown interest in a company's products/services     |
| Contact        | replace | an individual person associated with an account or organization                                   |
| Campaign       | replace | marketing initiative or project designed to achieve specific goals, such as generating leads etc. |
| Product2       | replace | for managing and organizing your product-related data within the Salesforce ecosystem             |
| Pricebook2     | replace | used to manage product pricing and create price books                                             |
| PricebookEntry | replace | an object that represents a specific price for a product in a price book                          |
| Opportunity            | merge | represents a sales opportunity for a specific account or contact                                                            |
| OpportunityLineItem    | merge | represents individual line items or products associated with an opportunity                                                 |
| OpportunityContactRole | merge | represents the association between an Opportunity and a contact                                                             |
| Account                | merge | individual or organization that interacts with your business                                                                |
| CampaignMember         | merge | association between a contact or lead and a campaign                                                                        |
| Task                   | merge | used to track and manage various activities and tasks within the salesforce platform                                        |
| Event                  | merge | used to track and manage calendar-based events, such as meetings, appointments calls, or any other time-specific activities |

## Setup Guide

### Grab credentials

To set up your pipeline, you'll need your Salesforce `user_name`, `password`, and `security_token`. Use
your login credentials for user_name and password.

To obtain the `security_token`, follow these steps:

1. Log into Salesforce with your credentials.

1. Click your profile picture/avatar at the top-right.

1. Select "Settings" from the drop-down.

1. Under "Personal Setup" in the sidebar, choose "My Personal Information" > "Reset My Security
   Token".

1. Click "Reset Security Token".

1. Check your email for the token sent by Salesforce.

> Note: The Salesforce UI, which is described here, might change.
The full guide is available at [this link.](https://developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/quickstart_oauth.htm)


### Initialize the verified source

To get started with your data pipeline, follow these steps:

1. Enter the following command:

   ```bash
   dlt init salesforce duckdb
   ```

   [This command](../../reference/command-line-interface) will initialize
   [the pipeline example](https://github.com/dlt-hub/verified-sources/blob/master/sources/salesforce_pipeline.py)
   with Salesforce as the [source](../../general-usage/source) and
   [duckdb](../destinations/duckdb.md) as the [destination](../destinations).

1. If you'd like to use a different destination, simply replace `duckdb` with the name of your
   preferred [destination](../destinations).

1. After running this command, a new directory will be created with the necessary files and
   configuration settings to get started.

For more information, read the guide on [how to add a verified source.](../../walkthroughs/add-a-verified-source)

### Add credentials

1. Inside the `.dlt` folder, you'll find a file called `secrets.toml`, which is where you can
   securely store your access tokens and other sensitive information. It's important to handle this
   file with care and keep it safe. Here's what the file looks like:

   ```toml
   # put your secret values and credentials here. do not share this file and do not push it to github
   [sources.salesforce]
   username = "please set me up!" # Salesforce user name
   password = "please set me up!" # Salesforce password
   security_token = "please set me up!" # Salesforce security token
   ```

1. In `secrets.toml`, replace username and password with your Salesforce credentials.

1. Update the security_token value with the token you
   [copied earlier](salesforce.md#grab-credentials) for secure Salesforce access.

1. Next, follow the [destination documentation](../../dlt-ecosystem/destinations) instructions to
   add credentials for your chosen destination, ensuring proper routing of your data to the final
   destination.

For more information, read the [General Usage: Credentials.](../../general-usage/credentials)

## Run the pipeline

1. Before running the pipeline, ensure that you have installed all the necessary dependencies by
   running the command:
   ```bash
   pip install -r requirements.txt
   ```
1. You're now ready to run the pipeline! To get started, run the following command:
   ```bash
   python salesforce_pipeline.py
   ```
1. Once the pipeline has finished running, you can verify that everything loaded correctly by using
   the following command:
   ```bash
   dlt pipeline <pipeline_name> show
   ```
   For example, the `pipeline_name` for the above pipeline example is `salesforce`, you may also use
   any custom name instead.

For more information, read the guide on [how to run a pipeline](../../walkthroughs/run-a-pipeline).

## Sources and resources

`dlt` works on the principle of [sources](../../general-usage/source) and
[resources](../../general-usage/resource).

### Source `salesforce_source`:

This function returns a list of resources to load users, user_role, opportunity,
opportunity_line_item, account etc. data from Salesforce API.

```python
@dlt.source(name="salesforce")
def salesforce_source(
    user_name: str = dlt.secrets.value,
    password: str = dlt.secrets.value,
    security_token: str = dlt.secrets.value,
) ->Iterable[DltResource]:
```

- `user_name`: Your Salesforce account username.

- `password`: Corresponding Salesforce password.

- `security_token`: Token for Salesforce API authentication, configured in ".dlt/secrets.toml".

### Resource `sf_user` (replace mode):

This resource function retrieves records from the Salesforce "User" endpoint.

```python
@dlt.resource(write_disposition="replace")
def sf_user() -> Iterator[Dict[str, Any]]:
    yield from get_records(client, "User")
```

Besides "sf_user", there are several resources that use replace mode for data writing to the
destination.

| user_role() | contact() | lead() | campaign() | product_2() | pricebook_2() | pricebook_entry() |
|-------------|-----------|--------|------------|-------------|---------------|-------------------|

The described functions fetch records from endpoints based on their names, e.g. user_role()
accesses the "user_role" endpoint.

### Resource `opportunity` (incremental loading):

This resource function retrieves records from the Salesforce "Opportunity" endpoint in incremental
mode.

```python
@dlt.resource(write_disposition="merge")
def opportunity(
    last_timestamp: Incremental[str] = dlt.sources.incremental(
        "SystemModstamp", initial_value=None
		)
) -> Iterator[Dict[str, Any]]:

    yield from get_records(
        client, "Opportunity", last_timestamp.last_value, "SystemModstamp"
    )
```

`last_timestamp`: Argument that will receive [incremental](../../general-usage/incremental-loading) state, initialized with "initial_value".
It is configured to track "SystemModstamp" field in data item returned by "get_records" and then yielded.
It will store the newest "SystemModstamp" value in dlt state and make it available in "last_timestamp.last_value" on next pipeline run.

Besides "opportunity", there are several resources that use replace mode for data writing to the
destination.

| opportunity_line_item() | opportunity_contact_role() | account() | campaign_member() | task() | event() |
|-------------------------|----------------------------|-----------|-------------------|--------|---------|

The described functions fetch records from endpoints based on their names, e.g.,
opportunity_line_item() accesses the "opportunity_line_item" endpoint.

## Customization

### Create your own pipeline

If you wish to create your own pipelines, you can leverage source and resource methods as discussed
above.

To create your data pipeline using single loading and
[incremental data loading](../../general-usage/incremental-loading), follow these
steps:

1. Configure the pipeline by specifying the pipeline name, destination, and dataset as follows:

   ```python
   pipeline = dlt.pipeline(
       pipeline_name="salesforce_pipeline",  # Use a custom name if desired
       destination="duckdb",  # Choose the appropriate destination (e.g., duckdb, redshift, post)
       dataset_name="salesforce_data"  # Use a custom name if desired
   )
   ```

   To read more about pipeline configuration, please refer to our
   [documentation](../../general-usage/pipeline).

1. To load data from all the endpoints, use the `salesforce_source` method as follows:

   ```python
   load_data = salesforce_source()
   source.schema.merge_hints({"not_null": ["id"]}) #Hint for id field not null
   load_info = pipeline.run(load_data)
   # print the information on data that was loaded
   print(load_info)
   ```

   > A hint ensures that the id column is void of null values. During data loading, dlt will verify
   > that the source's id column doesn't contain nulls.

1. To use the method `pipeline.run()` to load custom endpoints “candidates” and “members”:

   ```python
   load_info = pipeline.run(load_data.with_resources("opportunity", "contact"))
   # print the information on data that was loaded
   print(load_info)
   ```

   In the initial run, the "opportunity" and "contact" endpoints load all data using 'merge' mode
   and 'last_timestamp' set to "None". In subsequent runs, only data after
   'last_timestamp.last_value' (from the previous run) is merged. Incremental loading is specific to
   endpoints in merge mode with the “dlt.sources.incremental” parameter.

   > For incremental loading of endpoints, maintain the pipeline name and destination dataset name.
   > The pipeline name is important for accessing the
   > [state](../../general-usage/state) from the last run, including the end date
   > for incremental data loads. Altering these names could trigger a
   > [“full_refresh”](../../general-usage/pipeline#do-experiments-with-full-refresh),
   > disrupting the metadata tracking for
   > [incremental data loading](../../general-usage/incremental-loading).

1. To load data from the “contact” in replace mode and “task” incrementally merge mode
   endpoints:

   ```python
   load_info = pipeline.run(load_data.with_resources("contact", "task"))
   # pretty print the information on data that was loaded
   print(load_info)
   ```

   > Note: In the referenced pipeline, the "contact" parameter is always loaded in "replace" mode,
   > overwriting existing data. Conversely, the "task" endpoint supports "merge" mode for
   > incremental loads, updating or adding data based on the 'last_timestamp' value without erasing
   > previously loaded data.

<!--@@@DLT_SNIPPET_START tuba::salesforce-->
<!--@@@DLT_SNIPPET_END tuba::salesforce-->
# dbt_artifacts (Visivo Plugin)

Generate dashboards that deliver immediate insights into your dbt project by visualizing the metadata tables from the `dbt_artifacts` package. This Visivo plugin provides a quick way to monitor and analyze your dbt runs, model performance, test results, and data freshness – all within Visivo's YAML-based BI-as-code environment.

## Introduction

**Visivo dbt_artifacts** is a pre-built dashboard plugin for [Visivo](https://visivo.io) (an open-source, YAML-based BI-as-code tool). Its purpose is to enable rapid visibility into your dbt project’s runs and artifacts by leveraging the **`dbt_artifacts`** library from Brooklyn Data Co. The `dbt_artifacts` package builds a mart of tables and views describing your dbt project and its run history. By including this plugin in your Visivo project, you can instantly generate charts and dashboards that surface key metrics from those artifact tables (such as run durations, model execution times, test failures, and source freshness). This helps you achieve **dbt observability** without having to build dashboards from scratch.

**What is `dbt_artifacts`?** It is a community-supported dbt package by Brooklyn Data Co. that captures dbt run metadata. With every dbt run, dbt produces JSON artifacts (e.g. manifest, run_results, sources) containing information about your project and execution results. The `dbt_artifacts` package parses these artifacts and materializes them into database tables for analysis. In short, *“This package builds a mart of tables and views describing the project it is installed in.”* These tables include details on each run invocation, each model/test execution, and the current state of your dbt project. (You can find the `dbt_artifacts` package on the [dbt Hub](https://hub.getdbt.com/brooklyn-data/dbt_artifacts/latest/) and on [GitHub](https://github.com/brooklyn-data/dbt_artifacts) for reference.)

**How this plugin works:** Once your dbt project is set up to produce the `dbt_artifacts` tables, the Visivo plugin can be included via a YAML include. It comes with predefined Visivo traces/charts that query the artifact tables and assemble insightful dashboards. This allows you to quickly answer questions like: *“Which models are slowing down my runs?”*, *“How have run times changed over the last month?”*, *“Which tests are frequently failing?”*, or *“Are my sources fresh?”* – all through configurable, live dashboards in Visivo.

## Getting Started (Setup & Installation)

To start using the **dbt_artifacts plugin** in your Visivo project, you need to ensure two things: (1) your dbt project is producing the `dbt_artifacts` tables, and (2) your Visivo project includes this plugin. Below are the steps to set up everything:

1. **Install the `dbt_artifacts` package in your dbt project** – In your dbt project’s `packages.yml`, add the Brooklyn Data `dbt_artifacts` package (version 2.9.3 or latest), then run `dbt deps` to install it. For example:

   ```yaml
   # packages.yml
   packages:
     - package: brooklyn-data/dbt_artifacts
       version: 2.9.3   # use the latest stable version
   ```

   Ensure your dbt version is compatible (the package requires dbt >= 1.3.0).

2. **Configure dbt to capture run results** – Add an **on-run-end hook** in your `dbt_project.yml` to upload run results to the artifact tables after each run. This is critical for populating the facts about each invocation and model/test execution. For example, add the following:

   ```yaml
   # dbt_project.yml
   on-run-end:
     - "{{ dbt_artifacts.upload_results(results) }}"
   ```

   This macro, provided by the package, takes the `results` of each run and uploads them to your warehouse. (It’s often recommended to wrap this in a conditional to run only in production runs.) After adding the hook, run a full dbt build or at least one `dbt run` in your project so that the artifact tables are created and populated. You can also manually run `dbt run --select dbt_artifacts` to build all artifact models.

3. **Verify the artifact tables** – After running dbt, you should have a set of new tables in your warehouse (or data platform) corresponding to the dbt artifacts. The plugin expects these tables to exist and be populated with your run history. *(See **“Supported dbt_artifacts Models”** below for the list of tables.)* By default, the `dbt_artifacts` package will create these tables in your target database/schema (unless you configured a custom database/schema for it). Make sure your Visivo project can access this same target (e.g., via your data connection settings).

4. **Include the Visivo plugin in your project** – In your Visivo `project.visivo.yml` (or equivalent main YAML), use Visivo’s **includes** feature to pull in this repository. Add an entry under the `includes` section pointing to this repository. For example:

   ```yaml
   # project.visivo.yml
   includes:
     - path: visivo-io/dbt_artifacts.git@main
   ```

   This will fetch the dbt_artifacts plugin (using the `main` branch of this repo) into your Visivo project. You can pin to a specific tag or commit as needed once versions are available. After including, all the plugin’s predefined **traces** (SQL queries) and **charts** (visualizations) become available in your project. In many cases, the plugin also provides ready-to-use dashboard definitions, or you can reference the included charts in your own dashboards.

5. **(Optional) Configure plugin settings** – If your `dbt_artifacts` tables reside in a different data warehouse or have a different connection profile than your default, you may need to adjust your Visivo project configuration. For example, you might set an environment variable or Visivo variable to point to a specific target/connection name (similar to how other Visivo plugins work). By default, the plugin will use the primary target of your project for queries. (If needed, for instance, you could define something like `DBT_ARTIFACTS_TARGET` in your environment and have the YAML reference it – this plugin can be extended to support such configuration in the future.) In most cases, if your dbt artifact tables are in the same database as your analytics data, no extra configuration is required.

6. **Run Visivo and explore** – Launch your Visivo application (e.g., `visivo serve`) and navigate to the dashboards or charts provided by the plugin. You should see new dashboards (or the ability to create dashboards using the plugin’s charts) that give you insights into your dbt runs. The sections below describe what visualizations are included.

## Supported dbt_artifacts Models

The **dbt_artifacts** package produces a number of tables (models) in your warehouse. This Visivo plugin is designed to work with the core set of these artifact tables (as of `dbt_artifacts` v2.9.x). The expected models include both fact tables that record events from each dbt run, and dimension tables that describe the objects in your dbt project. For clarity, here are the key tables and what they represent:

- **Fact Tables** (one record per execution event in each run):
  - `fct_dbt__invocations` – Each dbt invocation (run) is logged here, including run timestamp, duration, status (success/failure), and contextual info (e.g. environment, invoked by, etc). This is essentially a history of all dbt runs.
  - `fct_dbt__model_executions` – Each model execution within each run. Contains the model name, run invocation ID, execution time, status (success or error), and whether it was skipped or executed.
  - `fct_dbt__test_executions` – Each test execution from each run. This covers both data tests and schema tests (and can include source freshness checks if those are run as tests). It logs whether tests passed or failed in each run.
  - `fct_dbt__seed_executions` – Each seed operation in each run (if you use `dbt seed` for CSV data). Logs similar info (time, status) for seed loads.
  - `fct_dbt__snapshot_executions` – Each snapshot operation in each run (if you use `dbt snapshot`). Logs snapshot execution details per run.

- **Dimension Tables** (metadata about dbt project elements):
  - `dim_dbt__models` – Metadata for each model in your dbt project (e.g. model name, path, materialization, tags, etc).
  - `dim_dbt__tests` – Metadata for each test in your project (test name, associated model/source, severity, etc).
  - `dim_dbt__sources` – Metadata for each source defined in your project (source name, freshness criteria if any, etc).
  - `dim_dbt__seeds` – Metadata for each seed file in your project.
  - `dim_dbt__snapshots` – Metadata for each snapshot in your project.
  - `dim_dbt__exposures` – Metadata for each exposure in your project (if you use exposures to represent downstream dependencies or reports).
  - `dim_dbt__current_models` – A convenience table representing the **current state** of models in your project (it often reflects the latest manifest information, such as whether a model is currently present/active).

Note: The plugin assumes these tables are present and up-to-date. If you upgrade the dbt_artifacts package or add new models (for example, future versions might introduce new artifact tables), the plugin may require updates to support them. (Always re-run the dbt_artifacts models after upgrading the package to ensure new fields/tables are created
raw.githubusercontent.com.) For detailed documentation on each model, refer to the official dbt_artifacts doc. 

## Dashboards & Visualizations in the Plugin

This plugin comes with a set of pre-built **dashboards**, **charts**, and **traces** that cover various aspects of your dbt project's performance and health. Once included, you can use these dashboards out-of-the-box or incorporate the charts into your own custom dashboards. Below are the key dashboards/visualizations that the plugin provides or supports:

- **Run History Overview** – A dashboard showing the history of dbt runs (from `fct_dbt__invocations`). This typically includes a timeline or bar chart of recent runs with their statuses (success/failure), run durations, and counts of models or tests run. You can quickly see which runs failed and how run times are trending over time. For example, a line chart of run duration by date, or a bar chart of models built per run.

- **Model Performance** – Visualizations highlighting how long each model takes to run and how often they fail. For instance, the plugin might include a table or bar chart of the slowest models (average or latest runtime from `fct_dbt__model_executions`), or a distribution of model run times. This helps identify bottlenecks in your DAG – e.g., which models are consistently slow or which runs had outlier slow models.

- **Test Results & Failures** – Dashboards focused on test execution outcomes (from `fct_dbt__test_executions`). These could show the number of test failures per run, highlight specific tests that are frequently failing, or show the proportion of tests passing vs failing over time. For example, a chart might list the tests with the most failures in the last 30 days, or display a time series of total failing tests per run.

- **Source Freshness** – If you are using dbt source freshness checks (which populate the `sources.json` artifact and appear in `fct_dbt__test_executions` as special tests), the plugin can visualize source freshness status. For instance, a table could list sources that failed freshness checks and when they last were updated. A dashboard might also summarize how many sources are fresh vs stale in the latest run.

- **Project Metadata Summary** – Using the dimension tables (like `dim_dbt__models`, `dim_dbt__sources`, etc.), the plugin might include charts that give an overview of your project composition. For example, a pie chart of models by materialization type (incremental vs view vs table), or a count of models per tag or owner. You could also list basic stats such as total number of models, tests, and sources.

## Customization and Extensibility

One of the advantages of using Visivo and this plugin is that everything is code-based and extensible. You can **customize** the provided dashboards or build new ones on top of the `dbt_artifacts` data to suit your needs. Here are some ways to customize or extend the plugin:

- **Using charts in your own dashboards**: Reference the plugin’s traces and charts directly in your project’s YAML. For example, if the plugin provides a trace named `Longest Model Run Times`, you can include it via `ref("Longest Model Run Times")` in your own dashboard definitions.

- **Customizing queries**: Copy trace definitions into your local project and modify the SQL to adjust filters, add new metrics, or change aggregation windows.

- **Theming and layout**: Combine the plugin’s charts with your custom visuals in a dashboard layout that matches your branding or narrative flow.

- **Extending to new metrics**: Build additional traces on artifact tables (or new tables/models you add) to surface metrics like build scheduling statistics, environment comparisons, or alert thresholds.

- **Configuration**: While the plugin defaults to your primary data target, you can extend it to accept configuration variables (e.g., via environment variables) for connecting to different schemas or environments.

## Roadmap

We have many ideas to further improve the `dbt_artifacts` plugin. Here are a few items on our **roadmap** for future development:

- **Additional Visualizations**: Detailed timing breakdowns (e.g., Gantt charts of runs to visualize the critical path), test coverage dashboards, and model dependency graphs with performance overlays.

- **dbt Cloud Integration**: Charts that leverage dbt Cloud metadata (job names, triggers, run reasons) for richer job-level reporting.

- **Alerting/Thresholds**: Built-in visual alerts or notifications for long-running models, repeated test failures, or stale sources.

- **Configurable Filters and Parameters**: Global UI controls or YAML variables to filter dashboards by environment, date range, or tags without editing SQL.

- **Compatibility and Updates**: Keep pace with updates in the `dbt_artifacts` package and Visivo itself, ensuring new fields and visualization types are supported.

- **Documentation and Examples**: Add screenshots, sample data scenarios, and a demo project to illustrate plugin usage.

## Contributing

Contributions are very welcome! If you have ideas for new charts, find a bug, or want to enhance the plugin, please follow these steps:

1. **Open an Issue** – Describe the problem or propose a feature in a new issue.
2. **Fork and Develop** – Fork the repository, implement changes in your fork, and test locally.
3. **Submit a Pull Request** – Open a PR with your changes. Please include context, screenshots (if applicable), and any relevant tests.
4. **Review Process** – We’ll review your PR and collaborate on improvements.

Please refer to a future `CONTRIBUTING.md` for detailed guidelines.

## License

This project is open source under the **MIT License**. See the [LICENSE](./LICENSE) file for details.

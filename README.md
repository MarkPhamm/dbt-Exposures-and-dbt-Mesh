# dbt-Exposures
Define and describe downstream uses of your dbt project

Exposure are define under the ```exposure``` key under the YAML file. Usually under 
```models/exposures/```

```yaml
# models/exposures.yml

version: 2

exposures:
  - name: Orders_data
    label: Orders_data
    type: notebook
    maturity: high
    url: https://tinyurl.com/jaffle-shop-reporting
    description: 'Expsosure for orders metrics'
    depends_on:
      - ref('fct_orders')
      - metric('order_total')
      - metric('large_orders')
      - metric('cumulative_order_amount_mtd')
    owner:
      name: Michael McData
      email: data@jaffleshop.com
  
  - name: Customers_data
    label: Customers_data
    type: notebook
    maturity: high
    url: https://tinyurl.com/jaffle-shop-reporting
    description: 'Exposure for customer metrics'
    depends_on:
      - ref('dim_customers')
    owner: 
      name: Data Startrek
      email: operations_manager@enterprise.com

```

Key components of this exposure:
1. `name`: Identifier for the exposure
2. `label`: Human-readable name for the exposure
3. `type`: What kind of exposure this is (dashboard, application, etc.)
4. `maturity`: The maturity level of the exposure
5. `owner`: Who maintains this exposure
6. `depends_on`: Which DBT models and metrics this exposure relies on
7. `description`: Business purpose of the exposure
8. `meta`: Additional metadata about the exposure
9. `url`: The URL where the exposure can be accessed

This YAML file would typically be placed in your `models` directory alongside your model definitions. The exposure helps document downstream uses of your data models and creates lineage in your documentation.

By defining exposures, you establish a clear lineage from your raw data to its downstream end-use cases. Defining exposures allows you to:

- **Visualize** the downstream resources and processes in your global lineage.
- **Run, test, and list** resources that feed into your exposure.
- **Populate** a dedicated page in your documentation (on the docs site and in Explorer) with context relevant to data consumers.

Exposures are configured in `.yml` files inside your `models/` directory. Once configured, an exposure will appear as a node in your lineage graph. Exposures can reference models, metrics, and sources, although we advise against referencing sources in exposures.

- To run all nodes upstream of an exposure, use the command: `dbt run --select +exposure:my_exposure_name`
- To run all nodes upstream of all exposures, use the command: `dbt run --select +exposure:*`

You can view exposures in the docs site after running the command `dbt docs generate`. Additionally, you can view exposures configured in your main branch in dbt Explorer after successfully running a production job.

# dbt Mesh  

## 1. Monotholic Data Structure vs dbt Mesh

### 1.1 **The challenge:** Collaboration breaks down at scale
* Reduced Trust:  
    - Unexpected issues due to changes made by upstream contributors  
    - Lack of standardization in development practices  

* Reduced Shipment Speed:  
    - Central / platform team becomes a bottleneck  
    - Onboarding new users becomes time-consuming and difficult  
    - Challenging to find correct models to work on  

* Increased Costs:  
    - Elongated development cycles  
    - Redundant data products, leading to higher spend  
    - Limited visibility on data platform spend by domain  

### 1.2 **The solution**: dbt Mesh 
dbt Mesh enable domain level ownership of data - without compromising on governance or creating silos

Definition: dbt Mesh is a decentralized data management architecture. In a data Mesh framework, the team not only own the data but also own the data pipeline and process of transforming it

**Benefits for Teams**  
* Domain Teams Get:  
    - Autonomy to ship data products faster  
    - Ability to share with and reference from other teams  
    - Confidence that nothing breaks unexpectedly  

* Central Data Teams Get:  
    - Visibility into the full lineage of transformations  
    - Running in a single platform (dbt)  
    - Without acting as a bottleneck for every domain team  

**Manage Interface with model governance**
- Every model can have a **contract** that defines guarantees around data shape.  
- Models can have **private** or **public access levels**, as well as be **grouped**. Public models can be reused across projects.  
- When a model’s contract changes in a way that’s backwards incompatible, it should be reflected with a new **version**.  

**Example:**  
- **Particle Error**  
  - While `model.mp_dist_project_mother_model` attempted to reference `code.model.mp_dist_project.txt_trans-label_history`, which is not allowed because the referenced code is private to the `github` group.  

## 2. Model Governance

### 2.1 Model Contracts

#### 2.1.1 **Model Contracts Defined**
- Allow you to *guarantee* the shape of your model  
  - The columns & names that exist in the model  
  - The data type of each column  
  - If your model is materialized as **table** or **incremental** (depending on the platform)  

- If the model doesn't have those exact columns with those exact data types, you'll get an error message when you try to do a dot run.

#### 2.1.2 **Creating a Model Contract**
To create a Model Contract, add a `contract:` configuration to the model YAML:

1. List each expected column along with its data type  
2. Add a `constraints` property to create stricter contracts  

Example:  
```yaml
models:
  - name: best_trilogy
    group: Legions_of_the_Southlands
    config:
      contract:
        enforced: True
    columns:
      - name: number_in_trilogy
        data_type: float
        constraints:
          - type: not_null
      - name: film_name
        data_type: string
```

#### 2.1.3 **Model Contracts vs. Testing**
- **Model contracts** verify the *structure of a dataset* (columns, data types), while **tests** validate the *content of a dataset*.

- **Tests offer more flexibility for content validation**:
  - Any query-based validation can be implemented as a test.
  - Configurable severity levels and custom thresholds simplify debugging.
  - **Limitation**: Tests run post-load; "bad data" may already exist in the model before detection.

- **Key difference**:
  - **Constraints** (in contracts) act as *pre-flight checks*: Block invalid data from entering the model if enforced by the platform.
  - **Tests** are *post-hoc checks*: Identify issues after data is loaded.

#### 2.1.4 **Model Contract Summary**
Model contracts guarantee the number of columns, their names, and data types for a given table.

With model contracts, you can:
- Ensure the data types of your columns.
- Ensure a model will have a desired shape.
- Apply constraints to your model. (When applying constraints, your data platform will perform additional validations on data as it is being populated. Check our docs to see which constraints your data platform supports.)

---

### 2.2 Model Versions
Model versions allow you to introduce breaking changes without disrupting downstream models immediately.

**Why use model versions?**
- Safely test prerelease changes.
- Promote new versions as the source of truth.
- Provide a migration window from older versions.

---

#### 2.2.1 Implementing Model Versions

##### 2.2.1.1 Create the new model file
- Example: `model_v2.sql`

##### 2.2.1.2 Add `latest_version` to the model’s YAML

```yaml
models:
  - name: dim_customers
    latest_version: 2
    columns:
      - name: customer_id
        data_type: number
```

##### 2.2.1.3 Define all versions under `versions:`

```yaml
models:
  - name: dim_customers
    latest_version: 2
    columns:
      - name: customer_id
        data_type: number
      - name: number_of_orders
        data_type: number
    versions:
      - v: 1
      - v: 2
```

##### 2.2.1.4 Add column configurations per version

```yaml
versions:
  - v: 1
  - v: 2
    columns:
      include: *
      exclude: number_of_orders
```

---

#### 2.2.2 Managing Aliases

##### 2.2.2.1 Default behavior
Each version creates a relation like `<model_name>_v<version>`.

##### 2.2.2.2 Custom alias (optional)

```yaml
versions:
  - v: 1
    config:
      alias: dim_customers
```

---

#### 2.2.3 Ref-ing & Running Model Versions

##### 2.2.3.1 Ref a specific version

```sql
select * from {{ ref('dim_customers', v=2) }}
```
- If no version is specified, it defaults to the **latest**.

##### 2.2.3.2 Run models

- Run all versions:  
  ```bash
  dbt run --select dim_customers
  ```

- Run a specific version:  
  ```bash
  dbt run --select dim_customers_v2
  ```

- Run the latest version:  
  ```bash
  dbt run -s dim_customers:version:latest
  ```

---

#### 2.2.4 Full YAML Example

```yaml
models:
  - name: file_name
    latest_version: 2
    columns:
      - name: column_name
        data_type: its_data_type
      - name: a_different_column_name
        data_type: that_columns_data_type
    versions:
      - v: 2
      - v: 1
        defined_in: file_name_v1
        config:
          alias: file_name
        columns:
          - include: *
            exclude: a_different_column_name
          - name: a_different_column_name
            data_type: that_columns_data_type
```

---

### 2.3 Group and Access Modifiers


#### 2.3.1 Groups 
* **Model Groups:** Models that are related to one another and owned by a specific team — a way for a business to organize models based on ownership (e.g., finance, marketing, employee data, etc.) rather than stage of development (e.g., staging, intermediate, etc.)
* **Model Access Modifiers:** Which models can access (reference) a specific model

Example:
```yml
groups:
  - name: finance
    owner:
      name: Firstname Lastname
      email: finance@jaffleshop.com
      slack: finance-data
      github: finance-data-team

  - name: product
    owner:
      email: product@jaffleshop.com
      github: product-data-team

```

#### 2.3.2 Access Modifiers
* Access modifiers determine which models can access (reference) a specific model
* There are three levels of model access that can be granted:

  * **Private**

    * Can only be accessed by those in the same group
    * Upstream models that need to be further transformed before they’re ready for downstream consumption

  * **Protected**

    * Can only be accessed by those in the same project/package
    * By default, all models are ‘protected’
    * This means that other models in the same project can reference them
    * Also upstream models

  * **Public**

    * Anyone can access
    * Will enable multi-project collaboration
    * Ready for downstream consumption
    * Probably a marts model or a staging model referenced in other projects

A **public** model or a **protected** model should be one that is guaranteed to be **ready for final use**.

A **private** model should be one that you don’t really want other people pulling from. For whatever reason, the data is a private implementation detail reserved for use only by the team represented by the group, and **there are probably downstream models better suited to be pulled by others**.

**Example**

To assign **Access Modifiers** to a model, add the `access:` property to the model YAML file.
Indicate the level of access you want to assign to this model:

* `private` / `protected` / `public`

```yaml
models:
    - name: model_name
         group: group_name
         access: access_modifer #public, private, or protecte
```

## 3. Multi-Project Collaboration

### 3.1 Intro to Multi-Project Collaboration
* Manage interfaces between contributors
* Collaborate across multiple dbt projects
* Intuitively navigate and explore dbt projects
* Provide contributors the flexibility to develop the way they’re most comfortable

### 3.2 Cross-Project Ref
Contracts, versions, and access levels all serve the purpose of helping data teams collaborate well even in larger, more complex projects. But even with these in place, there are still other problems introduced by large scale:

* It's hard for any contributor to have an intimate understanding of the entire dbt DAG
* It's hard to find the right models to work on
* It's hard to empower teams to work autonomously

An enterprise dbt Cloud account is required to create multiple dbt projects.

* To reference a model in another project, include a `dependencies.yml` in the downstream project listing the upstream project as a dependency:

  ```yaml
  projects:
    - name: name_of_project
  ```

* To reference a model from another project, use the cross-project `ref` macro:

  ```jinja
  {{ ref('project_name', 'model_name') }}
  ```

* Using a cross-project `ref` macro enables multi-project lineage, which is viewable in the **Explorer** tab.
  Multi-project lineage allows you to switch between lineage graphs of your projects.

### 3.3 Cross-Project Orchestration
Triggering a job on job completion will allow you to break free of brittle time-based dependencies and break up long-running jobs into multiple parts, improving debugging and resiliency. 

To enable a job in a downstream project to trigger on completion of a job in an upstream project, toggle 'Run when another job finishes' on and select the desired job from the upstream project.
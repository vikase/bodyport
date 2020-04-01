# Bodyport Data Challenge

--------------------
1. Data Organization
--------------------

    Bodyport:

    This dataset must be accessible across multiple data teams. How would you organize the raw data to make it available as efficiently as possible?

    Design a schema that would allow you to access subject information and raw data.

    This dataset was recently provided by a clinic of subjects who have recently come in for check ups. In the near future, another dataset will be provided that may or may not contain the same subjects and
    needs to be organized with this first dataset provided. Ensure your implementation can handle any future data coming in.

        Sepehr:

        There are 3 distinct needs implied here:

        1) Data Lake -- a filesystem to store raw, structured, and unstructured data blobs of any type, neatly organized for programmatic access
        2) Data Catalog -- crawled metadata aout the directory structure, loaded into a relational database, because filesystem searches are inefficient
        3) Data Warehouse -- reconciling new data with old in a heavily normalized relational database. Includes processed data elements.

        In /notebooks/, I demonstrate that by using a Data Warehouse/Catalog hybrid we can standardize efficient
        access to raw and metadata across teams for analytics and ML purposes.

        Access to datawarehouse tables is standardized via a thin but flexible and powerful ORM (object-relational mapping)
        defined in `bodyport/orm.py`. By installing the bodyport package, all teams can use a shared toolset to promote
        access. The efficiency gains of using a shared toolset in any organization is, in my experience, enormous
        and a key to reaching the famed "Plateau of Productivity".

        The `DataWarehouseManager()` class in `bodyport/load.py` serves as both a Catalog Manager as well as a
        DW manager -- it crawls the filesystem for metadata, and can store the results of analyses or pointers to them on
        the filesystem/S3. It's capable of incrementally loading and reconciling data.

        I've implemented a small command line utility in this package to demonstrate its use. By installing this package
        via `pip install -e .` you will have access to the `bodyport` command line interface. Simply run
        `bodyport dw demo` in the terminal for a demo of incremental, idempotent loads.

        Most of my explanations of reasoning are in the comments of the code in the DataWarehouseManager class
        and the accopmaning notebook in `./notebooks/demo_data_warehouse_operation.ipynb`

        ---------------------

        More details on the above concepts below:

        **1. THE DATA LAKE**

        Goal: enable safe programmatic access to raw data in a well-organized directory structure

        S3 is often the tool of choice here for storing raw incoming data due to its inexpensiveness and excellent quality.

        Data Lake design starts with organizing data in such a way as to anticipate likely changes to your incoming data patterns.

        I may recommend the following top-level S3 bucket for this dataset:

        ``s3://bodyport-data-lake/incoming/clinic={clinic_id}/measurement=ecg/latest/`` -- effectively, this is the bucket that clinics in question
        will have access to in order to upoad their latest files.

        Let's go over each part of the URL:

        `bodyport-data-lake` - This is the parent bucket for all of Bodyport's raw data -- unique in all of S3.

        `/incoming/` - indicates this bucket will contain data uploaded from outside sources and is not processed

        `/clinic={clinic_id}/` - I'm adding the additional anticipation of partnering with new clinics or entities to receive data.
        This way this clinic can have its own bucket. We'll call the current clinic `sf_state` in this example.
        The `clinic=` part making it easy to programmatically extract the clinic ID knowing only the run path, .

        `/measurement=ecg/` is included to anticipate other biomarker data in the future.

        `/latest/` Best practice for incoming data is for the provider to always upload into the same bucket, and our internal
        orchestrator (e.g. Airflow or AWS Pipeline) will listen for new data, and immediately move them into a newly created bucket,
        then deleting the content of `/latest/`.

        NB: This data structure is reproduced locally in this package under `./data`. However, I make the assumption
        that data has already been moved out of `/latest/` into a timestamped folder:

        data received for this assignment:
        `data/incoming/clinic=sf_state/measurement=ecg/2020-01-01`

        data I added to simulate incoming future data:
        `bodyport/data/incoming/clinic=sf_state/measurement=ecg/2020-12-01/`

        The existing data structure makes "lookup" operations perfectly efficient-- i.e. given Subject X and Run Y, any team can
        programmatically fetch the raw data by constructing a URL:

        `s3://bodyport-data-lake/incoming/clinic={clinic_id}/measurement=ecg/latest/subject_{X}/run_{Y}.csv`

        and

        `s3://bodyport-data-lake/incoming/clinic={clinic_id}/measurement=ecg/latest/subject_{X}/run_{Y}_header.json`

        Of course, searching the filesystem is slow and inefficient, particularly in a cloud environment, for just about any other query type than a basic lookup.

        This limitation is addressed by maintaining a data catalog.

        2) THE DATA CATALOG:

        To enable our team and others to ask more complex questions about the data we have, we'll want to crawl the filesystem periodically,
        generate some metadata about the data we have, and put that metadata in a database. Then we can apply all of the magic of SQL
        to systematically query our filesystem and ensure integrity of our pipelines.

        We can then answer
        1) How many runs do we get per upload from clinic X?
        2) Which runs have come in after the last time we populated the data warehouse?

        At a minimum, we could use a single table called `run_metadata`:

        +-----------------+-----------+
        | raw_path        | VARCHAR   |
        +-----------------+-----------+
        | last_crawled_at | TIMESTAMP |
        +-----------------+-----------+
        | processed_at    | TIMESTAMP |
        +-----------------+-----------+

        Since the data organization is rather simple in this example, I have assigned the responsibilities
        that typically are assigned to a data catalog to the Data Warehouse

        3) THE DATA WAREHOUSE

        Reconciling new data with old is one of the jobs of a Data Warehouse.

        The Data Warehouse can also host engineered features and transformed data elements.

        For example, age and sex are provided in Run metadata (header.json files), and ideally we would like these
        at the  "subject" level so we can quickly answer questions like:

        1) How old is subject X?
        2) WHat's the average age of subjects?
        3) How many male and female subjects do we have?
        4) And, if we have done advanced feature engineering in, for example, python: Does the average heartbeat of men and women differ?

        Our ETL solution extracts this information and load into 2 tables:
        - `subject`
        - `run`

        Since SQL is not powerful enough to usefully analyze ECG timeseries data anyway, I would recommend not storing
        it in the Data Warehouse. It is the largest part of the dataset by size, databases are expensive, and preserving data-types
        from python (which is loosely typed) to databases (which are strongly typed) can become a pain.

        Lower dimensional representations and statistical summaries, such as average beats-per-minute, etc.
        can be stored in the database, however. It is a very useful tool for sharing analyses.
        For instance, I've added a run_hash to uniquely identify a run by MD5 hashing the content of the raw file.
        This signature can serve to prevent duplicates from entering the data warehouse from the data provider.


2. Data aggregation:​
    The organized data needs to be able to get queried for technical and non-technical purposes.
    Describe the tools you would create in order to query the structured dataset.
    Implement two of these tools to query the data.

        Sepehr:
        This package implements an ORM to standardize access to the Data Warehouse. It can
        easily be modified to use a cloud storage backend rather than local.

        The metadata can be easily queried via SQL, either using a python DBAPI (sqlalchemy)
        or in a database viewer like DBeaver. I've also implemented a quick helper function
        for retrieving the raw data from disk/cloud via the ORM.

        ```python
        from bodyport.orm import Run, create_session

        session = create_session()

        sample_run = session.query(Run).filter_by(subject_id=1, run_id=1).first()

        run_df = sample_run.raw   # this method fetches the CSV and reads it as a dataframe
        ```

        Arbitrary SQL on the DataWarehouse is facilitated by the DataWarehouseManager:

        ```python
        from bodyport.load import DataWarehouseManager

        dw = DataWarehouseManager()

        dw.pandas_query("SELECT count(*) FROM subject WHERE birth_year > 1960")
        ## returns e.g. count=38
        ```

3. Data preprocessing:​
    The raw data may require some level of preprocessing to make it easier to analyze.
    What methods would you use to clean the signals?
    Implement your method to produce a filtered set of signals.
    Organize the filtered data according to your implemented data schema from part 1.

        Sepehr:
        Data preprocessing using Fourier based methods to address baseline drift and noise reduction
        is available in [the eda (exploratory data analysis) notebook](./notebooks.eda.ipynb)

4. Data interpretation and visualization:​
    Describe some of the key information contained in this filtered data.
    For instance, what are some prominent features that have been revealed in each time series that might be useful
    for further analysis and model development?
    How would you visualize this data? What plotting techniques would you use for this data set?

        Sepehr:
        Data nterpretation and visualization work is in [the eda (exploratory data analysis) notebook](./notebooks.eda.ipynb)


5. Data modeling:​
    How would you approach the question: “How can I distinguish between different individuals given only their ECG data?”
    Consider if there is any variation across an individual’s records, or across individuals that may be used.

        Sepehr:
        We would want to come up with some low-dimensional representations of an individual time series to discuss





Credits
-------

This package was created with Cookiecutter_ and the `audreyr/cookiecutter-pypackage`_ project template.

.. _Cookiecutter: https://github.com/audreyr/cookiecutter
.. _`audreyr/cookiecutter-pypackage`: https://github.com/audreyr/cookiecutter-pypackage

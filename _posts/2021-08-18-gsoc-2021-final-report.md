---
layout: post
title: Final report for GSoC'21
gh-repo: robbat2/rgw-s3-coverage-testing
cover-img: /assets/img/2020-08-16/cover.png
thumbnail-img: /assets/img/2020-08-16/thumb.png
tags: [ gsoc, ceph, RGW, S3, coverage]
---

## Google Summer of Code 2021 Project
- **Title**: RGW: S3 SDK Compatibility
- **Project link**: [Take me there!](https://summerofcode.withgoogle.com/projects/#5755795897057280)
- **Mentors**: Robin H. Johnson &lt;[robbat2](https://github.com/robbat2)>  & Ali Maredia &lt;[alimaredia](https://github.com/alimaredia)>
- **Organization**: [Ceph](https://ceph.com/en/)
- **Repository**: [RGW S3 Coverage testing](https://github.com/robbat2/rgw-s3-coverage-testing)

## Introduction
As it stands today, [s3-tests](https://github.com/ceph/s3-tests) use a limited fraction of the Boto S3 functionality. By instrumenting code coverage of AWS Boto SDK and s3-tests, gaps in s3-tests can be identified. The `objective` of this project is to identify parts of unused S3 source code of AWS SDKs using code coverage tools and consequently facilitate writing compatibility tests in s3-tests that cover those portions of the SDK for better coverage.

## Contents
- [Getting started](#getting-started)
- [Coverage output](#coverage-output)
- [Project Goals](#project-goals)
  - [Automating s3tests and ceph cluster with coverage](#automating-s3tests-and-ceph-cluster-with-coverage)
  - [Generating coverage output files](#generating-coverage-output-files)
  - [Analyzing coverage of the source SDK files](#analyzing-coverage-of-the-source-sdk-files)
- [Pull Request changelog](#pull-request-changelog)
- [Work left To Do (Future Work)](#work-left-to-do\(future-work\))
- [Credits](#special-thanks)

## Getting started
Firstly, to get started clone this [repository](https://github.com/robbat2/rgw-s3-coverage-testing).
Then, to build the RGW `s3-tests` testing environment with coverage follow the instructions below:

``Build using Scripts (stable)``: The script **bootstrap** starts a [ceph-demo](https://github.com/ceph/ceph-container/blob/master/src/daemon/demo.sh) cluster in a container and also an [s3-tests](https://github.com/ceph/s3-tests) container against the RGW of the ceph-demo cluster. It also automates generating `coverage` reports `(JSON+XML+HTML)` and a XML output of s3-tests that was run (nose-output.xml). 

A sample configuration file named ``s3tests.conf.SAMPLE`` has been provided in this repo which serves as the configuration file for running s3tests. Make changes in the ``.SAMPLE`` file itself, boostrap script would generate corresponding ``.conf`` file for you with appropriate configurations.

The scripts takes in arguments of the section of s3-tests to run along with nosetests.
The boostrap script already includes the prefix ``
S3TEST_CONF=your.conf ./virtualenv/bin/nosetests -v``.The corresponding section of tests to run that comes after the above command is provided as argument to the bootstrap script.

For example, if you want to run all the s3-tests like this:
```
S3TEST_CONF=your.conf ./virtualenv/bin/nosetests
```
Then simply run:
```
./bootstrap
```
If you want to specify which directory of tests to run like this:
```
S3TEST_CONF=your.conf ./virtualenv/bin/nosetests s3tests.functional
```
Then run:
```
./bootstrap s3tests.functional
```
If you want to specify which file of tests to run like this:
```
S3TEST_CONF=your.conf ./virtualenv/bin/nosetests s3tests.functional.test_s3
```
Then run:
```
./bootstrap s3tests.functional.test_s3
```
If you want to specify which test to run like this:
```
S3TEST_CONF=your.conf ./virtualenv/bin/nosetests s3tests.functional.test_s3:test_bucket_list_empty
```
Then run:
```
./bootstrap s3tests.functional.test_s3:test_bucket_list_empty
```
To gather a list of tests being run, run this:
```
./bootstrap --collect-only
```

``Build using Docker-compose (WIP)``: To start the ceph cluster and the s3-tests container simply run the following command to bootstrap both the services in docker-compose. It's in-between container networking portion is still WIP, features to be added.
```
docker-compose up -d
```
*Note: You can modify the `.conf.SAMPLE` according to your requirements and the `bootstrap` script will generate the corresponding `.conf` file internally.

## Coverage output
Leveraging the above build methods with specified tests, an output directory will appear by the name of `s3tests-output/`. This directory contains the following files:
- `cov-analysis.txt`: Contains a list of all Boto SDK s3 source files with corresponding function signatures that has less than 100% coverage. Read more here.
- `report.txt`: Contains a tabular representation of the coverage (hits and misses) of the source files against the test.
- `coverage_html/`:  This directory contains all the annotated coverage html files for easy viewing of coverage output corresponding to the test ran.
- `coverage.json` : This JSON file contains the coverage output of BOTO SDK and other source files corresponding to test taht was run.
- `coverage.xml`: This XML file contains the coverage output of BOTO SDK and other source files corresponding to test taht was run.
- `nose-output.xml`: This XML file contains the output of the tests that was run against the RGW for easy debugging pruposes. 

## Project Goals
The project work during the GSoC period can be divided into three parts:

### Automating s3tests and ceph cluster with coverage
In the first phase of the project, we automated the running of [s3-tests](https://github.com/ceph/s3-tests) with coverage tools against the RGW of the ceph-demo cluster in a dockerized setup. The coverage library used is the [coverage.py](https://coverage.readthedocs.io/en/coverage-5.5/) python package. In this portion we created the Dockerfile and the `run-tests.sh` script for running s3-tests with coverage and bootstrapping the ceph-demo cluster with the necessary configuration as mentioned in the `.conf.SAMPLE` file. The `host_IP` in the `.conf.SAMPLE` is set accordingly to the persistent container IP of the ceph-demo cluster in the system.

### Generating coverage output files 
In this phase of the project, we refactored our scripts and docker related files to scope in the feature of generating coverage output files for specific tests and attributed suite of nosetests in the [s3-tests](https://github.com/ceph/s3-tests) repo in all three formats,  `HTML`, `JSON`, `XML`. The `nose-output.xml` was also added for debugging purposes.

### Analyzing coverage of the source SDK files
In the last phase of project, we explored and implemented a Python script (a.k.a [`analyzer.py`](https://github.com/robbat2/rgw-s3-coverage-testing/blob/main/analyzer.py)) which analyzes the `coverage.json` file and identifies the portions of the Boto SDK source files that needs coverage and lists out the sourc file signatures with the corresponding source-file paths in the `cov-analysis.txt` file against the test that was run against the RGW.

**Aim**
From the generated coverage reports (XML,JSON) we want to find the exact function definitions in Boto SDK source files which were hit when a specific test was run against the RGW.

**Idea**: 
We extract the information from coverage output obtained for the Boto SDK source files. To that end, we parse the `coverage.json`. A snippet of it is provided below:

```json
{
"files": {
        "/s3-tests/s3tests_boto3/functional/policy.py": {
                    "executed_lines": [
                        1,
                        3,
                        4,
                        11,
                        23,
                        24,
                        27,
                        31,
                        40
                    ],
                    "summary": {
                        "covered_lines": 9,
                        "num_statements": 26,
                        "percent_covered": 30.0,
                        "missing_lines": 17,
                        "excluded_lines": 0,
                        "num_branches": 4,
                        "num_partial_branches": 0,
                        "covered_branches": 0,
                        "missing_branches": 4
                    },
                    "missing_lines": [
                        5,
                        6,
                        7,
                        8,
                        9,
                        12,
                        18,
                        19,
                        21,
                        25,
                        28,
                        29,
                        32,
                        38,
                        44,
                        45,
                        46
                    ],
                    "excluded_lines": []
          }
 }
```

With each source file attribute under ``files`` we checked whether it was a Boto SDK source file and then consequently we extracted the `missing_lines` list and other coverage measures for each such source s3 SDK file. The next step was to identify from the `missing_lines` list the function signatures in the corresponding source files which needs more coverage. 

**Solution approach**
1. We set the Boto SDK source files' python function signatures as an `Interval tree` ( each interval spans the starting line number as lower limit and function end line number as upper limit).

2. When run against the exact line number of coverage output, we can compute exact function coverage of function definitions through the interval tree and identify the corresponding list of Boto SDK source-file function signatures where new tests need to be written for better coverage.

A snippet of the output of the `cov-analysis.txt` for the test `s3tests_boto3.functional.test_s3:test_bucket_acl_default` is given below:


```
...
URL:/s3-tests/virtualenv/lib/python3.6/site-packages/boto3/s3/inject.py
FUNCTION:object_summary_load
FUNCTION:upload_file
FUNCTION:download_file
FUNCTION:bucket_upload_file
FUNCTION:bucket_download_file
FUNCTION:object_upload_file
FUNCTION:upload_fileobj
FUNCTION:bucket_upload_fileobj
FUNCTION:object_upload_fileobj
FUNCTION:download_fileobj
FUNCTION:bucket_download_fileobj
FUNCTION:object_download_fileobj
...
```

Another snippet of the output of the `cov-analysis.txt` for the test `s3tests.functional.test_s3.test_append_normal_object` is given below:

```
...
URL:/s3-tests/virtualenv/lib/python3.6/site-packages/boto/s3/bucketlistresultset.py
FUNCTION:bucket_lister
FUNCTION:versioned_bucket_lister
FUNCTION:multipart_upload_lister
FUNCTION:object_download_fileobj
...
URL:/s3-tests/virtualenv/lib/python3.6/site-packages/boto/s3/connection.py
FUNCTION:check_lowercase_bucketname
FUNCTION:wrapper
FUNCTION:get_bucket_server
FUNCTION:build_url_base
FUNCTION:build_host
FUNCTION:build_auth_path
FUNCTION:build_path_base
...
```  
&nbsp;
## Pull Request changelog
The pull requests introduced in the project and their activity during the period of GSoC is summarized below:

                                                                                             
| **PR Title**                                                                                 | **Reviewer** | **Review Status** | **Merge Status** |
|:--------------------------------------------------------------------------------------------:|:------------:|:-----------------:|:----------------:|
| [Coverage analyzer script](https://github.com/robbat2/rgw-s3-coverage-testing/pull/9)        |  robbat2     | Approved          | Closed          |
| [Coverage reports](https://github.com/robbat2/rgw-s3-coverage-testing/pull/8)                |  robbat2     | Approved          | Closed             |
| [Bootstrap script features](https://github.com/robbat2/rgw-s3-coverage-testing/pull/6)       |  robbat2     | Approved          | Closed            |
| [Cluster in exited state](https://github.com/robbat2/rgw-s3-coverage-testing/pull/5)         |  robbat2     | Approved          | Closed           |
| [Automated coverage setup](https://github.com/robbat2/rgw-s3-coverage-testing/pull/4)        |  robbat2     | Approved          | Closed           |
| [refactor:Cluster exited and automation](https://github.com/rubygems/rubygems.org/pull/2438) |  robbat2     | Approved          | Closed            |
| [Docker setup refactor](https://github.com/robbat2/rgw-s3-coverage-testing/pull/2)           |  robbat2     | Approved          | Closed           |
| [enhancement:Coverage analyzer](https://github.com/robbat2/rgw-s3-coverage-testing/pull/10)  |  robbat2     | Approved          | Closed            |
| [Docker setup](https://github.com/robbat2/rgw-s3-coverage-testing/pull/1)                    |  robbat2     | Approved          | Closed            |
| [Docker setup](https://github.com/robbat2/rgw-s3-coverage-testing/pull/1)                    |  robbat2     | Approved          | Closed           |
| [key limit in delete_object()](https://github.com/ceph/s3-tests/pull/400)                    |  alimaredia  | Pending           | Open          |
| [Invalid iso8601 in transition date set in Transition rule ](https://github.com/ceph/s3-tests/pull/399)                    |  alimaredia  | Approved           | Closed        |


## Work left To Do (Future Work)
The project objectives was accomplished keeping in line with the Python SDK of AWS (Boto) and the results needs to extended toward SDKs in other languages like Java and Golang. A plan needs to be formulated going forward that align the coverage work with Ceph's maintained s3-tests repos in these programming languages. 
    
## Special Thanks
Overall it was a wonderful experience working first time in open-source and that too being coupled with the ever-growing Ceph community throughout the GSoC project. I was lucky enough to be mentored by very seasoned community contributors and more importantly very helping people in Robin H. Johnson &lt;[robbat2](https://github.com/robbat2)>  & Ali Maredia &lt;[alimaredia](https://github.com/alimaredia)>,
giving constructive suggestions and guiding me through the program.

Last but not the least, my final thanks to Google for organizing this amazing program. I feel GSoC made it easier for me to get started with open source contributions and development in general. It was personally a very exciting moment for me to realize that my code was gonna be used by the wider community. This idea made me a more responsible developer and brought out changes in me overall.
I look forward to continue contributing to the amazing Ceph community and open-source in general.

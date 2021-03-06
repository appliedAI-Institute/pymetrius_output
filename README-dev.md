# pymetrius_output development guide

This repository contains the pymetrius_output python library together with utilities for building, testing, 
documentation and configuration management. 

This project uses the [black](https://github.com/psf/black) source code formatter
and [pre-commit](https://pre-commit.com/) to invoke it as a Git pre-commit hook. We also use 
[isort](https://github.com/PyCQA/isort) for sorting 
imports and [nbstripout](https://github.com/kynan/nbstripout) to prevent outputs of notebooks to be committed.

When first cloning the repository, run the following command (after
setting up your virtualenv with dev dependencies installed, see below) to set up
the local Git hook:

```shell script
pre-commit install
```

## Local Development
Automated builds, tests, generation of docu and publishing are handled by CI/CD pipelines. 
You will find an initial version of the pipeline in this repo. Below are further details on testing 
and documentation. 

Before pushing your changes to the remote it is often useful to execute `tox` locally in order to
detect mistakes early on.

We strongly suggest to use some form of virtual environment for working with the library. E.g. with venv
(if you have created the project locally with the python-library-template, it will already include a venv)
```shell script
python -m venv ./venv
. venv/bin/activate
```
or conda:
```shell script
conda create -n pymetrius_output python=3.8
conda activate pymetrius_output
```
A very convenient way of working with your library during development is to install it in editable mode 
into your environment by running
```shell script
pip install -e .
```


### Additional requirements

The main requirements for developing the library locally are in `requirements-dev.txt`.
For building documentation locally (which is done as part of the tox suite) you will need pandoc. 
It can be installed e.g. via
```shell script
sudo apt-get update -yq && apt-get install -yq pandoc
```

### Testing and packaging
The library is built with tox which will build and install the package, run the test suite and build documentation.
Running tox will also generate coverage and pylint reports in html and badges. 
You can configure pytest, coverage and pylint by adjusting [pytest.ini](pytest.ini), [.coveragerc](.coveragerc) and
[.pylintrc](.pylintrc) respectively.

In order to facilitate testing without tox (which can be slow, especially when executed for the first time), the
steps executed by tox are available as bash scripts within the [build_scripts](build_scripts) directory. For example,
to run tests, execute notebooks in 4 parallel processes as integration tests (and docu) and build the docu, 
all without involving tox, you could run

```shell
pip install -r requirements-test.txt -r requirements-docs.txt
./build_scripts/run-all-tests-with-coverage.sh
./build_scripts/build-docs.sh
```

Concerning notebooks: all notebooks in the [notebooks](notebooks) directory will be executed during test run, 
the results will be added to the docu in the _Guides and Tutorials_ section. Thus, notebooks can be conveniently used
as integration tests and docu at the same time.

You can run thew build by installing tox into your virtual environment 
(e.g. with `pip install tox`) and executing `tox`. 

For creating a package locally run
```shell script
python setup.py sdist bdist_wheel
```

### Documentation
Documentation is built with sphinx every time tox is executed, doctests are run during that step.
There is a helper script for updating documentation files automatically. It is called by tox on build and can 
be invoked manually as
```bash
python build_scripts/update_docs.py
```
See the code documentation in the script for more details on that. We recommend using the bash script
```shell
./build_scripts/build-docs.sh
```
to both update the docs files and to rebuild the docu with sphinx in one command.

Notebooks also form part of the documentation, in case they have been rendered before (see explanation above).

## Configuration Management
If you decided to include configuration utils when generating the project from the template, this repository 
also includes [configuration utilities](config.py) that are often helpful when using data-related libraries. 
They are based on appliedAI's lightweight library [accsr](https://github.com/appliedAI-Initiative/accsr).

By default the configured secrets like access keys and so on are expected to be in a file called `config_local.json`.
In order for these secrets to be available in CI/CD during the build, _create a gitlab variable of type file called_
`CONFIG_LOCAL` containing your CI secrets. 
Note that sometimes it makes sense for them to differ from your own local config.

Generally the configuration utils support an arbitrary hierarchy of config files, you will have to adjust the
[config.py](config.py) and the gitlab pipeline if you want to make use of that.

## CI/CD and Release Process
This repository contains ci/cd pipelines for multiple providers. 
The most sophisticated one is the [gitlab ci pipeline](.gitlab-ci.yml) (this is what we use internally at appliedAI), it 
will run the test suite and publish docu, badges and reports. 
Badges can be accessed from the pipeline's artifacts, on gitlab the url of the coverage badge will be:

```
<gitlab_project_url>/-/jobs/artifacts/develop/raw/badges/coverage.svg?job=tox_use_cache
```

The azure ci pipeline is rather rudimentary, pull requests are always welcome!

### Development and Release Process with Gitlab and Github

In order to be able to automatically release new versions of the package, the
 CI pipeline should have access to the following variables / github secrets:

```
PYPI_REPO_USER
PYPI_REPO_PASS
```

They will be used in the release steps in the pipeline. If you want to publish packages to a private server,
you will also need to set the `PYPI_REPO_URL` variable.

On gitlab, you will need to set up a `Gitlab CI deploy key` and add it as file-type variable called `GITLAB_DEPLOY_KEY` 
for automatically committing from the develop pipeline during version bumping.

#### Automatic release process

In order to create an automatic release, a few prerequisites need to be satisfied:

- The project's virtualenv needs to be active
- The repository needs to be on the `develop` branch
- The repository must be clean (including no untracked files)

Then, a new release can be created using the `build_scripts/release-version.sh` script (leave off the version parameter
to have `bumpversion` automatically derive the next release version):

```shell script
./scripts/release-version.sh 0.1.6
```

To find out how to use the script, pass the `-h` or `--help` flags:

```shell script
./build_scripts/release-version.sh --help
```

If running in interactive mode (without `-y|--yes`), the script will output a summary of pending
changes and ask for confirmation before executing the actions.

#### Manual release process
If the automatic release process doesn't cover your use case, you can also create a new release
manually by following these steps:

1. (repeat as needed) implement features on feature branches merged into `develop`. 
Each merge into develop will advance the `.devNNN` version suffix and publish the pre-release version into the package 
registry. These versions can be installed using `pip install --pre`.
2. When ready to release: From the develop branch create the release branch and perform release activities 
(update changelog, news, ...). For your own convenience, define an env variable for the release version
    ```shell script
    export RELEASE_VERSION="vX.Y.Z"
    git checkout develop
    git branch release/${RELEASE_VERSION} && git checkout release/${RELEASE_VERSION}
    ``` 
3. Run `bumpversion --commit release` if the release is only a patch release, otherwise the full version can be specified 
using `bumpversion --commit --new-version X.Y.Z release` 
(the `release` part is ignored but required by bumpversion :rolling_eyes:).
4. Merge the release branch into `master`, tag the merge commit, and push back to the repo. 
The CI pipeline publishes the package based on the tagged commit.

    ```shell script
    git checkout master
    git merge --no-ff release/${RELEASE_VERSION}
    git tag -a ${RELEASE_VERSION} -m"Release ${RELEASE_VERSION}"
    git push --follow-tags origin master
    ```
5. Switch back to the release branch `release/vX.Y.Z` and pre-bump the version: `bumpversion --commit patch`. 
This ensures that `develop` pre-releases are always strictly more recent than the last published release version 
from `master`.
6. Merge the release branch into `develop`:
    ```shell script
    git checkout develop
    git merge --no-ff release/${RELEASE_VERSION}
    git push origin develop
    ```
6. Delete the release branch if necessary: `git branch -d release/${RELEASE_VERSION}`
7. Pour yourself a cup of coffee, you earned it! :coffee: :sparkles:

## Useful information

Mark all autogenerated directories as excluded in your IDE. In particular docs/_build and .tox should be marked 
as excluded in order to get a significant speedup in searches and refactorings.

If using remote execution, don't forget to exclude data paths from deployment (unless you really want to sync them)

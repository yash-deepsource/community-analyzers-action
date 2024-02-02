# Community Analyzers Action

This repository is an attempt at harnessing the power of [Composite actions](https://docs.github.com/en/enterprise-cloud@latest/actions/creating-actions/about-custom-actions#composite-actions) to make reporting SARIF information easy for users of Community Analyzers to DeepSource.

The analyzers currently supported are:

- [Dart Analyze](src/dart-analyze/action.yml)
- [Kube Linter](src/kube-linter/action.yml)
- [CloudFormation Linter (cfn-lint)](src/cfn-lint/action.yml)
- [Slither](src/slither/action.yml)
- [Solhint](src/solhint/action.yml)

## Inputs

This action requires the following inputs:

- `ref`: The ref of the community-analyzers-action repository.
- `analyzer`: The name of the analyzer to run.
- `deepsource-dsn`: The DSN of the DeepSource organization.
- `path`: The path of the directory or file to scan. Default is ".".
- `output-file`: The path to the output file. Default is "./deepsource-report.sarif".
- `extra-data-json`: Extra data as stringified JSON to be passed to the analyzer. Default is "{}".

## Usage

To use this action, you need to specify it in your workflow file and provide the necessary inputs. Here's an example:

```yml
- name: Run community analyzer
  uses: yash-deepsource/community-analyzers-action@main
  with:
    ref: ${{ github.action_ref }}
    analyzer: "dart-analyze"
    deepsource-dsn: ${{ secrets.DEEPSOURCE_DSN }}
    path: "./src"
    output-file: "./deepsource-report.sarif"
    extra-data-json: '{"sdk": "beta"}'
```

## How does it work?

### Structure

There is a parent [action.yml](./action.yml) which is responsible for deciding which analyzer is being run. Every analyzer has a sub-folder within the [src folder](./src) containing its own action files which perform the analyzer specific steps.

### Behaviour

The action can set up and run only one of the Community Analyzers at a time. This is since it's much more configurable for a user to configure the action parallelly in case the user wants to report multiple SARIF reports in a single workflow.

The primary action file performs the following tasks:

1. Using the `ref` input, it checks out the repository itself at `./.community-analyzers-action`. This is to allow us to refer to the same version of the sub-action file as the action itself. There wasn't a way to automatically determine the ref for this action, and hence this workaround.
2. It goes through a chain of `if`s and determines which analyzer's sub-action file to run. It passes to it the path of dir or file to scan, output-file's name and a stringified JSON that contains analyzers specific args.
3. Refer to the sub-action's `action.yml` file in order to learn what each sub-action does. In short, each sub-action always attempts to do the following:
   1. Set up the required env to generate the SARIF report
   2. Generate the SARIF report to the given path
4. Once the respective sub-action has run, the action reports the generated SARIF report to DeepSource.

### Why `extra-data-json`?

GitHub has an inherent limit of 10 inputs on composite actions. Apart from this, specifying all inputs individually would make maintenance of the `action.yml` very cumbersome.

### Why not a Docker action or JavaScript action?

I preferred this approach as it allowed us to split the action file itself into smaller chunks. I don't think we would have been able to do the same if this was a Dockerfile, do hit me up if you have an idea around this. JavaScript actions can't interact with the environment, and thus in cases we would have to set up env via a non `setup-` action, it would become very hacky to do so.

## Caveats and things to do to improve it

- Ideally, we would want a short script in bash, JS, python, etc. to verify the args given in `extra-data-json`. This is something we should definitely look into, as we have restrictions such as `sdk` value of the `dart-analyze` we support. A script would be the most efficient way to program such validations.
- Currently, testing this workflow file is a bit cumbersome. We should look into setting automated tests for the same.
- Maintaining this might turn out a bit cumbersome depending on the frequency of changes to community analyzers; which doesn't seem super high atm.

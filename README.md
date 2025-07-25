# Freely-available-programming-books
List of Free Learning Resources In Many Languages
name: 'AwesomeBot Markdown Summary Report'
description: 'Composes the summary report using JSON results of any AwesomeBot execution'

inputs:
  ab-root:
    description: 'Path where AwesomeBot result files are written.'
    required: true
  files:
    description: 'A delimited string containing the filenames to process.'
    required: true
  separator:
    description: 'Token used to delimit each filename. Default: " ".'
    required: false
    default: ' '
  append-heading:
    description: 'When should append report heading.'
    required: false
    default: "false"
  write:
    description: 'When should append the report to GITHUB_STEP_SUMMARY file descriptor.'
    required: false
    default: "true"

outputs:
  text:
    description: Generated Markdown text.
    value: ${{ steps.generate.outputs.text }}

runs:
  using: "composite"

  steps:

    - name: Generate markdown
      id: generate
      # Using PowerShell
      shell: pwsh
      # sec: sanatize inputs using environment variables
      env:
        GITHUB_ACTION_PATH: ${{ github.action_path }}
        GITHUB_WORKSPACE: ${{ github.workspace }}
        # INPUT_<VARIABLE_NAME> is not available in Composite run steps
        # https://github.community/t/input-variable-name-is-not-available-in-composite-run-steps/127611
        INPUT_AB_ROOT: ${{ inputs.ab-root }}
        INPUT_FILES: ${{ inputs.files }}
        INPUT_SEPARATOR: ${{ inputs.separator }}
        INPUT_APPEND_HEADING: ${{ inputs.append-heading }}
      run: |
        $text = ""

        # Handle optional heading
        if ("true" -eq $env:INPUT_APPEND_HEADING) {
          $text += "### Report of Checked URLs!"
          $text += "`n`n"
          $text += "<div align=`"right`" markdown=`"1`">`n`n"
          $text +=   "_Link issues :rocket: powered by [``awesome_bot``](https://github.com/dkhamsing/awesome_bot)_."
          $text += "`n`n</div>"
        }

        # Loop ForEach files
        $env:INPUT_FILES -split $env:INPUT_SEPARATOR | ForEach {
          $file = $_
          if ($file -match "\s+" ) {
            continue # is empty string
          }
          $abr_file = $env:INPUT_AB_ROOT + "/ab-results-" + ($file -replace "[/\\]","-") + "-markdown-table.json"

          try {
            $json = Get-Content $abr_file | ConvertFrom-Json
          } catch {
            $message = $_
            echo "::error ::$message" # Notify as GitHub Actions annotation
            continue                  # Don't crash!!
          }

          $text += "`n`n"
          if ("true" -eq $json.error) {
            # Highlighting issues counter
            $SearchExp  = '(?<Num>\d+)'
            $ReplaceExp = '**${Num}**'
            $text += "`:page_facing_up: File: ``" + $file + "``     (:warning: " + ($json.title -replace $SearchExp,$ReplaceExp) + ")"
            # removing where ab attribution lives (moved to report heading)
            $text += $json.message -replace "####.*?\n","`n"
          } else {
            $text += ":page_facing_up: File: ``" + $file + "``     (:ok: **No issues**)"
          }
        }

        # set multiline output (the way of prevent script injection is with random delimiters)
        # https://docs.github.com/en/actions/using-workflows/workflow-commands-for-github-actions#multiline-strings
        # https://github.com/orgs/community/discussions/26288#discussioncomment-3876281
        $delimiter = (openssl rand -hex 8) | Out-String
        echo "text<<$delimiter" >> $env:GITHUB_OUTPUT
        echo "$text" >> $env:GITHUB_OUTPUT
        echo "$delimiter" >> $env:GITHUB_OUTPUT


    - name: Write output
      if: ${{ fromJson(inputs.write) }}
      shell: bash
      env:
        INPUT_TEXT: ${{ steps.generate.outputs.text }}
        INPUT_WRITE: ${{ inputs.write }}
      run: |
        echo "$INPUT_TEXT"     >> $GITHUB_STEP_SUMMARY

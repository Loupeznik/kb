# Powershell

## Process piped output in a custom function

The piped output is passed to the `process` block of the function.

```powershell

Function Write-PipedString {
    param (
        [Parameter(Mandatory = $true, ValueFromPipeline = $true)]
        [string]$input
    )

    process {
        Write-Host "Processing $input from the pipe"
    }
}

Function Write-PipedStringWithoutBindings {
    process {
        $input = $args[0]

        if ($null -eq $input) {
            $input = $_
        }

        Write-Host "Processing $input from the pipe"
    }
}

# First example
"Test" | Write-PipedString
# Result: Processing Test from the pipe

# Second example
"TestWithoutBindings" | Write-PipedStringWithoutBindings

Write-PipedStringWithoutBindings "TestWithoutBindings"

# Result: Processing TestWithoutBindings from the pipe
```

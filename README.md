PSHTMLTable
==============

This is an old set of functions used to generate HTML tables, and highlight cells within them on demand.

One of PowerShell�s major strong points is its ability to interface with a variety of technologies and data sources.  This makes it a great candidate for sending ad hoc notifications or generating HTML based reports.  The attached functions can be used to spice up various notifications, reports, or other HTML generated by PowerShell.

A quick example showing a standard SCOM alert, and a SCOM alert generated using PSHTMLTable functions.  The latter is a bit more helpful!

**Plain SCOM Alert**:

[![plain alert](/Media/scom_plain.png)](/Media/scom_plain.png)

**PSHTMLTable SCOM Alert**:

[![scom spicy](/Media/scom_spicy.png)](/Media/scom_spicy.png)

## Instructions

```powershell
# One time setup
    # Download the repository
    # Unblock the zip
    # Extract the PSHTMLTable folder to a module path (e.g. $env:USERPROFILE\Documents\WindowsPowerShell\Modules\)

    #Simple alternative, if you have PowerShell 5, or the PowerShellGet module:
        Install-Module PSHTMLTable

# Import the module.
    Import-Module PSHTMLTable    #Alternatively, Import-Module \\Path\To\PSHTMLTable

# Get commands in the module
    Get-Command -Module PSHTMLTable
```

## Examples

### Simple Example

This illustrates creating a webpage with several Process related stats, using all of the functions in PSHTMLTable

```powershell
#gather 20 events from the system log and pick out a few properties
    $events = Get-EventLog -LogName System -Newest 20 | select TimeGenerated, Index, EntryType, UserName, Message

#Create the HTML table without alternating rows, colorize Warning and Error messages, highlighting the whole row.
    $eventTable = $events | New-HTMLTable -setAlternating $false |
        Add-HTMLTableColor -Argument "Warning" -Column "EntryType" -AttrValue "background-color:#FFCC66;" -WholeRow |
        Add-HTMLTableColor -Argument "Error" -Column "EntryType" -AttrValue "background-color:#FFCC99;" -WholeRow

#Build the HTML head, add an h3 header, add the event table, and close out the HTML
    $HTML = New-HTMLHead
    $HTML += "<h3>Last 20 System Events</h3>"
    $HTML += $eventTable | Close-HTML

#test it out
    set-content C:\test.htm $HTML
    & 'C:\Program Files\Internet Explorer\iexplore.exe' C:\test.htm
```

[![example output](/Media/e2_events.png)](/Media/e2_events.png)

### Complex Example

This illustrates creating a webpage with several Process related stats, using all of the functions in PSHTMLTable

```powershell
#This example demonstrates using the New-HTMLHead, New-HTMLTable, Add-HTMLTableColor, ConvertTo-PropertyValue and Close-HTML functions

#get processes to work with
    $processes = Get-Process

#Build HTML header
    $HTML = New-HTMLHead -title "Process details"

#Add CPU time section with top 10 PrivateMemorySize processes.  This example does not highlight any particular cells
    $HTML += "<h3>Process Private Memory Size</h3>"
    $HTML += New-HTMLTable -inputObject $($processes | sort PrivateMemorySize -Descending | select name, PrivateMemorySize -first 10)

#Add Handles section with top 10 Handle usage.
$handleHTML = New-HTMLTable -inputObject $($processes | sort handles -descending | select Name, Handles -first 10)

    #Add highlighted colors for Handle count

        #build hash table with parameters for Add-HTMLTableColor.  Argument and AttrValue will be modified each time we run this.
        $params = @{
            Column = "Handles" #I'm looking for cells in the Handles column
            ScriptBlock = {[double]$args[0] -gt [double]$args[1]} #I want to highlight if the cell (args 0) is greater than the argument parameter (arg 1)
            Attr = "Style" #This is the default, don't need to actually specify it here
        }

        #Add yellow, orange and red shading
        $handleHTML = Add-HTMLTableColor -HTML $handleHTML -Argument 1500 -attrValue "background-color:#FFFF99;" @params
        $handleHTML = Add-HTMLTableColor -HTML $handleHTML -Argument 2000 -attrValue "background-color:#FFCC66;" @params
        $handleHTML = Add-HTMLTableColor -HTML $handleHTML -Argument 3000 -attrValue "background-color:#FFCC99;" @params

    #Add title and table
    $HTML += "<h3>Process Handles</h3>"
    $HTML += $handleHTML

#Add process list containing first 10 processes listed by get-process.  This example does not highlight any particular cells
    $HTML += New-HTMLTable -inputObject $($processes | select name -first 10 ) -listTableHead "Random Process Names"

#Add property value table showing details for PowerShell ISE
    $HTML += "<h3>PowerShell Process Details PropertyValue table</h3>"
    $processDetails = Get-process powershell_ise | select name, id, cpu, handles, workingset, PrivateMemorySize, Path -first 1
    $HTML += New-HTMLTable -inputObject $(ConvertTo-PropertyValue -inputObject $processDetails)

#Add same PowerShell ISE details but not in property value form.  Close the HTML
    $HTML += "<h3>PowerShell Process Details object</h3>"
    $HTML += New-HTMLTable -inputObject $processDetails | Close-HTML

#write the HTML to a file and open it up for viewing
    set-content C:\test.htm $HTML
    & 'C:\Program Files\Internet Explorer\iexplore.exe' C:\test.htm
```

[![example output](/Media/e1_process.png)](/Media/e1_process.png)

### Simple Pester Example
```powershell
$results = Invoke-Pester -show none -passthru

# Format cells where the value is greater than 0
$params = @{
    ScriptBlock = {$args[0] -gt 0}
}

# Create a summary table and add spaces into the column names for asthetics, adding colours for passed and failed tests
$summaryTable = $results | Select-Object `
     @{Name="Passed Count";Expression={$_.PassedCount}},
     @{Name="Failed Count";Expression={$_.FailedCount}},
     @{Name="Skipped Count";Expression={$_.SkippedCount}},
     @{Name="Pending Count";Expression={$_.PendingCount}},
     @{Name="Total Count";Expression={$_.TotalCount}} | New-HtmlTable |
    Add-HTMLTableColor -Argument "Failed" -Column "Failed Count" -AttrValue "background-color:#ffb3b3;" @params |
    Add-HTMLTableColor -Argument "Passed" -Column "Passed Count" -AttrValue "background-color:#c6ffb3;" @params

# Create a table containing the results
$resultsTable = $results |Select-Object -ExpandProperty TestResult |
    Select-Object Describe, Context, Name, Result |
     New-HtmlTable |
        Add-HTMLTableColor -Argument "Failed" -Column "Result" -AttrValue "background-color:#ffb3b3;" <# -WholeRow #> |
        Add-HTMLTableColor -Argument "Passed" -Column "Result" -AttrValue "background-color:#c6ffb3;" <# -WholeRow #>

#Compose the html adding headers
    $HTML = New-HTMLHead
    $HTML += "<h3>Post Build Test Summary</h3>"
    $HTML += $summaryTable
    $HTML += "<h3>Post Build Test Results</h3>"
    $HTML += $resultsTable | Close-HTML

#test it out
    set-content C:\test.html $HTML
    & 'C:\Program Files\Internet Explorer\iexplore.exe' C:\test.html
```

[![example output](/Media/pester_simple.png)](/Media/pester_simple.png)

### Simple Pester Example
```powershell
# Invoke pester and store the results so that they can be referenced multiple times
$results = Invoke-Pester -show none -passthru

# Format cells where the value is greater than 0
$params = @{
    ScriptBlock = {$args[0] -gt 0}
}

# Create a summary table and add spaces into the column names for asthetics, adding colours for passed and failed tests
$summaryTable = $results | Select-Object `
     @{Name="Passed Count";Expression={$_.PassedCount}},
     @{Name="Failed Count";Expression={$_.FailedCount}},
     @{Name="Skipped Count";Expression={$_.SkippedCount}},
     @{Name="Pending Count";Expression={$_.PendingCount}},
     @{Name="Total Count";Expression={$_.TotalCount}} | New-HtmlTable |
    Add-HTMLTableColor -Argument "Failed" -Column "Failed Count" -AttrValue "background-color:#ffb3b3;" @params |
    Add-HTMLTableColor -Argument "Passed" -Column "Passed Count" -AttrValue "background-color:#c6ffb3;" @params

#Compose the html adding headers
$HTML = New-HTMLHead
$HTML += "<h3>Post Build Test Summary</h3>"
$HTML += $summaryTable
$HTML += "<h3>Post Build Test Results</h3>"

# Create a seperate table for each describe block
foreach ($section in ($results |Select-Object -ExpandProperty TestResult | Select-Object Describe -Unique)) {
    # Add a title based on the descibe block
    $HTML += ("<h4>{0}</h4>" -f $section.Describe)
    $HTML += $results | Select-Object -ExpandProperty TestResult | Where-Object -FilterScript { $_.Describe -eq $section.Describe } |
        Select-Object Context, Name, Result |
        New-HtmlTable |
            Add-HTMLTableColor -Argument "Failed" -Column "Result" -AttrValue "background-color:#ffb3b3;" |
            Add-HTMLTableColor -Argument "Passed" -Column "Result" -AttrValue "background-color:#c6ffb3;"
}

$HTML += "" | Close-HTML

#test it out
    set-content C:\test.html $HTML
    & 'C:\Program Files\Internet Explorer\iexplore.exe' C:\test.html
```

[![example output](/Media/pester_advanced.png)](/Media/pester_advanced.png)

## Functions

PSHTMLTable includes several functions:

* ConvertTo-PropertyValue - Convert an object with various properties into an array of property,value pairs
* New-HTMLHead - Starts building HTML including internal style sheet (leaves body, html tags open)
* New-HTMLTable - Create an HTML table from one or more objects
* Add-HTMLTableColor - Colorize cells or rows in an HTML table, or add other inline CSS
* CloseHTML - Close out the body and html tags

## Notes

This was written during my early days with PowerShell - pull requests would be welcome!

For more details, stop by [the old blog post](http://ramblingcookiemonster.wordpress.com/2013/08/06/powershell-and-tables/) on this!
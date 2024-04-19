Office scripts uses US formatting when pasting and reading from cells
Leads to differences when country uses other date notations
See ISO 8601

Create date in memory, log and paste in cell

See Example of Excel DatetimeFormatInfo
https://learn.microsoft.com/en-us/javascript/api/office-scripts/excelscript/excelscript.datetimeformatinfo?view=office-scripts

```TS
/**
 * This script sets the value of a cell to a date string for January 2, 2023.
 * It writes the day or month first in the string based on system settings.
 */
function main(workbook: ExcelScript.Workbook) {
  // Get the first cell in the current worksheet.
  const cell = workbook.getActiveWorksheet().getCell(0,0);

  // Get the date format.
  const cultureInfo : ExcelScript.CultureInfo = workbook.getApplication().getCultureInfo();
  const systemDateTimeFormat : ExcelScript.DatetimeFormatInfo = cultureInfo.getDatetimeFormat();
  const shortDatePattern : string = systemDateTimeFormat.getShortDatePattern();

  // Determine if the date should start with the month or day.
  if (shortDatePattern.startsWith("m")) {
    cell.setValue("1/2/2023");
  } else {
    cell.setValue("2/1/2023");
  }
}
```

# Record setting cell to 2 January
```TS
function main(workbook: ExcelScript.Workbook) {
	let selectedSheet = workbook.getActiveWorksheet();
	// Set range A1 on selectedSheet
	selectedSheet.getRange("A1").setValue("1/2/2023");
}
```

# Using Javascripts built-in date object
Office scripts comes with some built-in objects of JavaScripts for example Date and Math
https://learn.microsoft.com/en-us/office/dev/scripts/develop/javascript-objects

```TS
function main(workbook: ExcelScript.Workbook) {
	let dateRange = workbook.getActiveWorksheet().getRange("A1")
	let date = new Date(2023, 0, 2)
	console.log(date.toLocaleDateString())
	dateRange.setValue(date.toLocaleDateString())
}
```

What happens when my dates provided are within localeDateString
Can be forced to used certain locale
See BCP 47 Language Tag
Months are 0-based

```TS
function main(workbook: ExcelScript.Workbook) {
	let dateRange = workbook.getActiveWorksheet().getRange("A1")
	let date = new Date(2023, 0, 2)
	console.log(date.toLocaleDateString('nl-NL'))
	dateRange.setValue(date.toLocaleDateString('nl-NL'))
}
```

```TS
function main(workbook: ExcelScript.Workbook) {
	// Setting cell A1 to a string value
	let dateRange_str = workbook.getActiveWorksheet().getRange("A1")
	let date_str = new Date(2023, 0, 2)
	// let date = new Date(Date.parse("2023-02-01T00:00:00"))
	let dtString_str = date_str.toLocaleDateString()
	console.log(dtString_str)
	dateRange_str.setValue(dtString_str)

	// Setting cell A2 to a date value
	let dateRange_dv = workbook.getActiveWorksheet().getRange("A2")
	let date_dv = new Date(2023, 0, 2)
	// let date = new Date(Date.parse("2023-02-01T00:00:00"))
	console.log(date_dv.toLocaleDateString())
	dateRange_dv.setValue(date_dv.toLocaleDateString())

}
```
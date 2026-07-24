# MSSL EBT System Data Message Model and Interface Definitions

![GitHub Release](https://img.shields.io/github/v/release/jon-bong/mssl-ebt-models)
[![NuGet Version (Emc.SewPlus.Nems.Models)](https://img.shields.io/nuget/v/Mssl.Ebt.Models.svg?style=flat-square)](https://www.nuget.org/packages/Mssl.Ebt.Models/)
![NuGet Downloads](https://img.shields.io/nuget/dt/Mssl.Ebt.Models)
![GitHub License](https://img.shields.io/github/license/jon-bong/mssl-ebt-models)

Data models of transmission messages and interface definitions of files and reports used by the _Electronic Business Transaction (EBT)_ system of the _Market Support Services Licensee (MSSL)_ for _Open Electricity Market (OEM)_ retailers in Singapore.

All data models and interface definitions in this package are based on the _Market Participant User Manual_ and _Secured File Transfer Protocol Kit_ found in the [Resources: Becoming a Licensed Electricity Retailer](https://www.openelectricitymarket.sg/retailer/resources) page of the Open Electricity Market website.

## 🚀 Key Features
- Classes and interfaces representing the data structure of EBT system transmission messages, data files and reports.
- Interfaces can be implemented to create custom data models to integrate with any file-processing library that deals with delimited data.
- Use of enumerations to represent object properties that take on a finite set of `string` values.

## 📦 Installation
Install the package via the NuGet Package Manager Console, the Nuget Package Manager UI, the .NET CLI or by adding a package reference.

### .NET CLI
```bash
dotnet add package Mssl.Ebt.Models.x.x.x.nupkg
```

### Package Manager
```powershell
Install-Package Mssl.Ebt.Models.x.x.x.nupkg
```

## 🛠️ Quick Start & Usage
This package contains classes that serialise data to send to the EBT system and deserialise messages received from the EBT system.

### Transaction Messages
Transaction messages exchanged between the EBT system and its system participants are in XML format.

For example, a "Validation Acknowledgement" message is received as:

```xml

<?xml version="1.0" encoding="UTF-8"?>
<ValidationAcknowledgement>
    <TransactionId>9300009819:168025</TransactionId>
    <Result>pass</Result>
</ValidationAcknowledgement>

```

The XML data can be deserialised into a `ValidationAcknowledgement` object:

```csharp
XmlSerializer xmlSerializer = new XmlSerializer(typeof(ValidationAcknowledgement));
using (StreamReader reader = new StreamReader("Validation Acknowledgement.xml"))
{
    ValidationAcknowledgement validationAcknowledgement = xmlSerializer.Deserialize(reader) as ValidationAcknowledgement;
}

```

### Data File Interface Definitions
The contents of a data file received from the EBT system are in _CSV_ format, and are contained in an XML message in two forms:
- as a `string` value of a child element or
- saved in a _CSV_ file, compressed into a _ZIP_ file and then encoded as a _Base64_ `string` value (as `CDATA`) of a child element

Consider an XML message containing the following compressed (indicated by the `Compressed` element with value _Y_) CSV data (indicated by the `ContentFormat` element) representing adjusted usage:

```xml

<DispatchData>
  <TransactionId>06734455</TransactionId>
  <ContentFormat>CSV</ContentFormat>
  <Compressed>Y</Compressed>
  <Data>
    <![CDATA[UEsDBBQACAgIAMxpkVkAAAAAAAAAAAAAAAAEAAAAZGF0YX3UvU0DQRCA0RyJHlzACe/8+HxL7MSB
    y3ADlvsXCAmBkXnJBvvdJU87c7ner7fd+fS+G2PEOrdtvr6MdR+5z15iGW+f99/nMn61RCu0Rjug
    rWhHtA1tosVQlEyIJmQTwgnphHhCPiGgkFBKKPl2JJQSSgmlhFJCKaGUUEqoJFQSKo6XhEpCJaGS
    UEmoJFQSagm1hFpCzQ0koZZQS6gl1E+ELn9373wYveXrs5+fHnfvv63QGu2AtqId0Ta0iRZDUTIh
    mpBNCCekE+IJ+YSAQkIpoeTbkVBKKCWUEkoJpYRSQimhklBJqDheEioJlYRKQiWhklBJqCXUEmoJ
    NTeQhFpCLaGWUD8R+gBQSwcI01Cg9AkBAAD4CgAAUEsBAhQAFAAICAgAzGmRWdNQoPQJAQAA+AoA
    AAQAAAAAAAAAAAAAAAAAAAAAAGRhdGFQSwUGAAAAAAEAAQAyAAAAOwEAAAAA]]>
  </Data>
</DispatchData>

```

To extract the data, the _Base64_ encoded string first needs to be decoded into a _ZIP_ file and then the CSV file extracted:
```csharp

XmlSerializer xmlSerializer = new XmlSerializer(typeof(DispatchData));
using (StreamReader reader = new StreamReader("SRLP Usage Data.xml"))
{
    // deserialize the XML message into a DispatchData object
    DispatchData dispatchData = xmlSerializer.Deserialize(reader) as DispatchData;
    try
    {
        if (dispatchData.Compressed == Mssl.Ebt.Models.Messages.YesNo.Yes)
        {
            // decode & deflate the compressed data
            string base64String = string.Join(Environment.NewLine, dispatchData.Data);
            File.WriteAllBytes("srlp-usage-data.zip", Convert.FromBase64String(base64String));
            ZipArchive zipArchive = ZipFile.OpenRead("srlp-usage-data.zip");
            Dictionary<string, string> dataFiles = zipArchive.Entries
                .ToDictionary(f => f.Name, f => Path.Combine(Directory.GetCurrentDirectory(), f.Name));
            foreach (KeyValuePair<string, string> dataFile in dataFiles)
            {
                // extract each file from the zip archive to the current working directory
                string localFile = $"{dataFile.Value}.{dispatchData.ContentFormat.ToString().ToLower()}";
                zipArchive.Entries.First(f => f.Name.Equals(dataFile.Key, StringComparison.OrdinalIgnoreCase))
                    .ExtractToFile(localFile, true);

                // read the extracted file and process the data
                ...
            }
        }
    }
    catch
    {
        ...
    }
    finally
    {
        ...
    }
}

```

#### CSV File Entries as Interfaces
Each of the following interfaces under the `Mssl.Ebt.Models.Messages.DataFiles` namespace represents a detail line in the respective CSV file:
- `IAmiUsageData`: Advanced Metering Infrastructure (AMI) Usage Data file or MDA Adjusted AMI Usage Data file.
- `IConsumerHistoryData`: Consumer History Data file.
- `IMarketCompanyUsageData`: Market Company Usage data file.
- `IMdaAdjustedUsageAccount`: MDA-adjusted usage account.
- `ISrlpUsageData`: Static Residential Load Profile (SRLP) Usage Data file or MDA Adjusted SRLP Usage Data file.

Each of the above interfaces defines the structure of the CSV data, allowing itself to be implmemented by a class using a file-processing library (e.g. _CsvHelper_, _FileHelpers_) to read and write the CSV data. For example, an implementation of the `ISrlpUsageData` using the _FileHelpers_ library:

```csharp

using FileHelpers;
using Mssl.Ebt.Models.Messages.DataFiles;
using MyNamespace.Converters; // definition of custom converter DecimalValueConverter

/// <summary>
/// Represents an entry in the Static Residential Load Profile (SRLP) Usage Data or MDA Adjusted SRLP Usage Data file.
/// </summary>
[DelimitedRecord(",")]
internal class SrlpUsageData : ISrlpUsageData
{
    #region Object properties.

    /// <summary>
    /// The date of reading for this entry in <c>dd/MM/yy</c> format.
    /// </summary>
    [FieldConverter(ConverterKind.Date, "dd/MM/yy")]
    public DateTime RecordDate { get; set; }

    /// <summary>
    /// The half-hourly interval number (1 to 48).
    /// </summary>
    [FieldConverter(ConverterKind.Byte)]
    public byte Interval { get; set; }

    /// <summary>
    /// Consising of 10 digits and 7 decimals, this is the Active value for a SRLP Usage Data, 
    /// or the MDA Active (MDA Adjusted) for a MDA Adjusted SRLP Usage Data.
    /// </summary>
    [FieldConverter(typeof(DecimalValueConverter), 7)]
    public decimal ActiveValue { get; set; }

    /// <summary>
    /// Consising of 10 digits and 7 decimals, this is the Adjusted Active value for a SRLP Usage Data, 
    /// or the TLF Adjusted Active (MDA Adjusted) for a MDA Adjusted SRLP Usage Data.
    /// </summary>
    [FieldConverter(typeof(DecimalValueConverter), 7)]
    public decimal AdjustedActiveValue { get; set; }

    #endregion

    #region Constructors.

    /// <summary>
    /// Creates a new instance of the <see cref="SrlpUsageData"/> class.
    /// </summary>
    public SrlpUsageData()
    {
        this.RecordDate = DateTime.MinValue;
        this.Interval = default(byte);
        this.ActiveValue = default(decimal);
        this.AdjustedActiveValue = default(decimal);
    }

    #endregion
}

```

The `FileHelpersEngine<SrlpUsageData>` object then reads the CSV file and returns a collection of `SrlpUsageData` objects (`List<SrlpUsageData>`).

Each of the following classes represents a CSV section in its respective data file:
- `AmiMeterUsageData`: Advanced Metering Infrastructure (AMI) usage data or MDA adjusted AMI usage data for a single metering point.
- `ConsumerMeterHistoryData`: Consumer History Data file.
- `SrlpMeterUsageData`: Static Residential Load Profile (SRLP) usage data or MDA adjusted SRLP usage data for a single metering point.

## 🚀 Target Frameworks
* **.NET:** Core 3.0, Core 3.1, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0
* **.NET Framework:** 3.5, 4.0, 4.5, 4.5.2, 4.6.1, 4.6.2, 4.7.2, 4.8, 4.8.1
* **.NET Standard:** 2.0, 2.1

## 🗺️ Roadmap
- **[] Planned:** Incorporate JSON serialisation and deserialisation.

_Have a feature request? Please open a [feature suggestion](#feedback-and-support)._

## 🤝 Feedback and Support
User feedbacks, bug reports and feature requests are welcome! Since the core codebase is private, please use the following channels to get in touch:
- Bug Reports & Feature Requests: Please open an issue directly on our GitHub Issues Tracker.
- Discussions and Questions: send me an [e-mail](##author-and-contact).

## 👨‍💻 Author and Contact
* **Maintainer:** Jonathan Bong
* **E-mail:** [jonbong1607@hotmail.com](mailto:jonbong1607@hotmail.com)
* [![LinkedIn](https://img.shields.io/badge/linkedin-%230077B5.svg?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/jonathan-bong-5a229840/)
* [![GitHub](https://img.shields.io/badge/github-%23121011.svg?style=for-the-badge&logo=github&logoColor=white)](https://github.com/jon-bong)

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

# MSSL EBT System Data Message Model and Interface Definitions

![GitHub Release](https://img.shields.io/github/v/release/jon-bong/mssl-ebt-models)
[![NuGet Version (Emc.SewPlus.Nems.Models)](https://img.shields.io/nuget/v/Mssl.Ebt.Models.svg?style=flat-square)](https://www.nuget.org/packages/Mssl.Ebt.Models/)
![NuGet Downloads](https://img.shields.io/nuget/dt/Mssl.Ebt.Models)
![GitHub License](https://img.shields.io/github/license/jon-bong/mssl-ebt-models)
[![API Documentation](https://shields.io)](./docs/index.html)

Data models of transmission messages and interface definitions of files and reports used by the _Electronic Business Transaction (EBT)_ system of the _Market Support Services Licensee (MSSL)_ for _Open Electricity Market (OEM)_ retailers in Singapore.

All data models and interface definitions in this package are based on the _Market Participant User Manual_ and _Secured File Transfer Protocol Kit_ found in the [Resources: Becoming a Licensed Electricity Retailer](https://www.openelectricitymarket.sg/retailer/resources) page of the Open Electricity Market website.

## 🚀 Key Features
- Strongly typed classes representing transaction messages, data file and report detail records used in the EBT system.
- Interfaces can be implemented to create custom data models to integrate with any file processing library working on delimited data.
- Use of enumerations to represent object properties that take on a finite set of `string` values.

## 📚 Technical Documentation
The API reference guide has been generated using Sandcastle. 

👉 **[Browse the API Reference Document](./docs/index.html)**

### What's Inside?
* **Namespaces & Classes:** Full code architecture breakdown.
* **Methods & Properties:** Detailed parameter and return value info.

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

## 🛠️ Usage
The majority of the strongly typed classes under the `Mssl.Ebt.Models.Messages` namespace in this package participate in the XML serialisation/deserialisation of transaction messages received from or submitted to the EBT system.

### Transaction Messages
Consider an incoming "Validation Acknowledgement" message received from the EBT system:

```xml

<?xml version="1.0" encoding="UTF-8"?>
<ValidationAcknowledgement>
    <TransactionId>930000XXXX:168XXX</TransactionId>
    <Result>pass</Result>
</ValidationAcknowledgement>

```

The message can be deserialised into a `ValidationAcknowledgement` object:

```csharp
XmlSerializer xmlSerializer = new XmlSerializer(typeof(ValidationAcknowledgement));
using (StreamReader reader = new StreamReader("Validation Acknowledgement.xml"))
{
    ValidationAcknowledgement validationAcknowledgement = xmlSerializer.Deserialize(reader) as ValidationAcknowledgement;
}

```

### Data File Interface Definitions
The contents of a data file received from the EBT system can be in either _XML_ or _CSV_ format, indicated by the value of the `ContentFormat` element of the XML message.

A data file can also be of a significant size and hence may be compressed using the _ZIP_ standard. A compressed data file is indicated by the value _Y_ in the `Compressed` element of the XML message.
The compressed data takes the form of a string encoded with the Base64 Content-Transfer-Encoding algorithm and saved as `CDATA` in the `Data` element.

Consider the following XML message in a file _SRLP Usage Data.xml_. The message contains a _CSV_-formatted data file in compressed form:

```xml

<DispatchData>
  <TransactionId>0673XXXX</TransactionId>
  <ContentFormat>CSV</ContentFormat>
  <Compressed>Y</Compressed>
  <Data>
    <![CDATA[UEsDBBQACAgIAMxpkVkAAAAAAAAAAAAAAAAEAAAAZGF0YX3UvU0DQRCA0RyJHlzACe/8+HxL7MSB
    y3ADlvsXCAmBkXnJBvvdJU87c7ner7fd+fS+G2PEOrdtvr6MdR+5z15iGW+f99/nMn61RCu0Rjug
    rWhHtA1tosVQlEyIJmQTwgnphHhCPiGgkFBKKPl2JJQSSgmlhFJCKaGUUEqoJFQSKo6XhEpCJaGS
    ...
    ...
    ...
    NTeQhFpCLaGWUD8R+gBQSwcI01Cg9AkBAAD4CgAAUEsBAhQAFAAICAgAzGmRWdNQoPQJAQAA+AoA
    AAQAAAAAAAAAAAAAAAAAAAAAAGRhdGFQSwUGAAAAAAEAAQAyAAAAOwEAAAAA]]>
  </Data>
</DispatchData>

```

To obtain the data, the _Base64_ encoded string is first decoded as a _ZIP_ file and then the CSV data file extracted:

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
                FileHelpersEngine<SrlpUsageData> engine = new FileHelpersEngine<SrlpUsageData>();
                List<SrlpUsageData> srlpUsageDataList = engine.ReadFromFile(localFile);
                ...
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

#### CSV File Entries Modelled As Interfaces
Each of the following interfaces under the `Mssl.Ebt.Models.Messages.DataFiles` namespace represents a detail line in the respective CSV file:
- `IAmiUsageData`: Advanced Metering Infrastructure (AMI) Usage Data file or MDA Adjusted AMI Usage Data file.
- `IConsumerHistoryData`: Consumer History Data file.
- `IMarketCompanyUsageData`: Market Company Usage data file.
- `IMdaAdjustedUsageAccount`: MDA-adjusted usage account.
- `ISrlpUsageData`: Static Residential Load Profile (SRLP) Usage Data file or MDA Adjusted SRLP Usage Data file.

Each interface defines the base structure of the CSV data, allowing itself to be implemented by a class set up to work with a file-processing library (e.g. _CsvHelper_, _FileHelpers_) to read and write CSV data.

For example, the following is an implementation of the `ISrlpUsageData` interface by a class that is set up to work with the _FileHelpers_ library:

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
        this.RecordDate = default(DateTime);
        this.Interval = default(byte);
        this.ActiveValue = default(decimal);
        this.AdjustedActiveValue = default(decimal);
    }

    #endregion
}

```

The `FileHelpersEngine<SrlpUsageData>` object then reads the CSV file and returns a collection of `SrlpUsageData` objects (`List<SrlpUsageData>`), shown in the earlier example.

#### CSV File Sections
Each of the following classes represents a CSV section (a group of delimited entries) in its respective data file:
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

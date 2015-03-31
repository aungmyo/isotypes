# Isotypes ISO8583 Message Translator #
Intended for integration with Apache Camel, this Java library provides a Spring XSD custom configuration for defining ISO8583 messages and the message translation utilities to create and parse messages.
This library does not address the transport aspects of working with the ISO8583 protocol (e.g., sync vs. async, wire codec), but rather should be stage in a Camel route, as, for example, marshalling and unmarshalling.

## Version 1.2 (Nov 2014) ##
The project has undergone a major refactor, and consists of three sub-projects:
| iso8583-core | ISO8583 message translation |
|:-------------|:----------------------------|
| iso8583-spring | Spring XSD for XML message schema beans |
| iso8583-camel | Camel data format |

The Core module how has its own DSL builder for programmatic message schema definition, as well as parsing Json/Yaml/HOCON schema definitions.

(see [Version 1.0.0-RC2 source archive)](https://drive.google.com/file/d/0B6MHA0-6L2igTy1iWVFPdVo2NDA/view?usp=sharing)

## Features ##
  * Declarative message definition via custom Spring XSD
  * Content Types supported: ASCII, EBCDIC, UTF-8, BCD (_or any JVM supported encoding_)
  * [Automatic type conversions](TypeConversions.md)
  * Primary, Secondary & Tertiary Hex and Binary Bitmaps supported
  * Track1 & Track2 data supported
  * [Custom Type Formatters](CustomTypeFormatters.md) (inc. overridden standard field formatters)
  * Field values can also be provided in name-keyed maps and bean objects
  * Message content reporting (see [Output](#Message_Output.md), below)
  * 85%+ [unit test coverage](http://wiki.isotypes.googlecode.com/hg/jacoco/index.html)
  * [Field Value Auto-generation](AutoGeneration.md)
  * OSGi bundled
  * [Camel Integration](CamelIntegration.md) (full **`camel-iso8583`** component coming soon)

## Example ##
[Sample Code](http://code.google.com/p/isotypes/source/browse/#hg%2Fsrc%2Ftest%2Fjava%2Forg%2Fseefin%2Fformats%2Fiso8583%2Fsamples)
### Message Schema ###
The message schema file is included in the Spring context, and can be used to configure and inject a [MessageFactory](http://wiki.isotypes.googlecode.com/hg/apidocs/org/seefin/formats/iso8583/MessageFactory.html) for a client to use
```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
            http://www.springframework.org/schema/beans/spring-beans-3.1.xsd
        http://seefin.org/schema/iso8583 
            http://seefin.org/schema/iso8583-1.0.xsd">

    <schema id="testMessages" xmlns="http://seefin.org/schema/iso8583"
            header="ISO015000077" bitmapType="hex" contentType="text" charset="ascii">
        <description>Bank messages</description>

        <message type="0200" name="Acquirer Payment Request">
            <field f= "2" name="cardNumber"     desc="Payment Card Number"         dim="llvar(40)" type="ns" />
            <field f= "3" name="processingCode" desc="Processing Code"             dim="fixed(6)"  type="n" />
            <field f= "4" name="amount"         desc="Amount, transaction (cents)" dim="fixed(12)" type="n" />
            <field f= "7" name="transDateTime"  desc="Transmission Date and Time"  dim="fixed(10)" type="date" />
            <field f="11" name="stan"           desc="System Trace Audit Number"   dim="fixed(6)"  type="n" />
            <field f="12" name="transTimeLocal" desc="Time, local transaction"     dim="fixed(6)"  type="time" />
            <field f="13" name="transDateLocal" desc="Date, local transaction"     dim="fixed(4)"  type="date" />
            <field f="32" name="acquierID"      desc="Acquiring Institution ID"    dim="llvar(4)"  type="n">0000</field>
            <field f="37" name="extReference"   desc="Retrieval Reference Number"  dim="fixed(12)" type="n" />
            <field f="41" name="cardTermId"     desc="Card Acceptor Terminal ID"   dim="fixed(16)" type="ans" />
            <field f="43" name="cardTermName"   desc="Card Acceptor Terminal Name" dim="fixed(40)" type="ans" />
            <field f="48" name="msisdn"         desc="Additional Data (MSISDN)"    dim="llvar(14)" type="n" />
            <field f="49" name="currencyCode"   desc="Currency Code, Transaction"  dim="fixed(3)"  type="n" />
            <field f="90" name="originalData"   desc="Original data elements"      dim="lllvar(4)" type="xn" optional="true" />
        </message>
    </schema>
</beans>

```

### Java Client ###
This example creates a ISO8583 message and populates its business fields from a bean object (Request), whose properties match the field names defined in the XML ISO8583 schema, above.

Further field values ('technical values') are added separately (now can also be added declaratively via the schema, see  [Field Value Auto-generation](AutoGeneration.md))
```
public void
sendMessage ( MTI type, PaymentRequestBean request)
	throws IOException, ParseException
{
	// instantiate request from business object
	Message message = factory.createFromBean ( type, request);
		
	// add fields used by the ISO8583 protocol/server
	Date dateTime = (new SimpleDateFormat("dd-MM-yyyy HH:mm:ss")).parse ( "01-01-2013 10:15:30");
	message.setFieldValue ( 3, 101010);    // processing code
	message.setFieldValue ( 7, dateTime);  // transmission date and time
	message.setFieldValue ( 11, 4321);     // trace (correlation) number
	message.setFieldValue ( 12, dateTime); // transaction time
	message.setFieldValue ( 13, dateTime); // transaction date
		
	// log the message content:
	for ( String line : message.describe ())
	{
		System.out.println ( "INFO: " + line);
	}
		
	// check the message is good-to-go:
	List<String> errors = message.validate ();
	if ( errors.isEmpty () == false)
	{
		throw new MessageException ( errors);
	}
		
	// write it to the dummy output stream:
	factory.writeToStream ( message, output);
}
```
### Message Output ###
```
INFO: MTI: 0200 (version=ISO 8583-1:1987 class=Financial Message function=Request origin=Acquirer) name: "Acquirer Payment Request" header: [ISO015000077] #fields: 14
INFO:  F#: Dimension:Type  Value (is-a)                          Name             Description
INFO:   2: VAR2 ( 40):ns   [1234********34] (CardNumber)         cardNumber      'Payment Card Number'
INFO:   3: FIXED(  6):n    [101010] (Integer)                    processingCode  'Processing Code'
INFO:   4: FIXED( 12):n    [10] (BigInteger)                     amount          'Amount, transaction (cents)'
INFO:   7: FIXED( 10):date [Tue Jan 01 10:15:30 GMT 2013] (Date) transDateTime   'Transmission Date and Time'
INFO:  11: FIXED(  6):n    [4321] (Integer)                      stan            'System Trace Audit Number'
INFO:  12: FIXED(  6):time [Tue Jan 01 10:15:30 GMT 2013] (Date) transTimeLocal  'Time, local transaction'
INFO:  13: FIXED(  4):date [Tue Jan 01 10:15:30 GMT 2013] (Date) transDateLocal  'Date, local transaction'
INFO:  32: VAR2 (  4):n    [3031] (Integer)                      acquierID       'Acquiring Institution ID'
INFO:  37: FIXED( 12):n    [1234] (Long)                         extReference    'Retrieval Reference Number'
INFO:  41: FIXED( 16):ans  [ATM-1234] (String)                   cardTermId      'Card Acceptor Terminal ID'
INFO:  43: FIXED( 40):ans  [BOI/ATM/D8/SJG] (String)             cardTermName    'Card Acceptor Terminal Name'
INFO:  48: VAR2 ( 14):n    [353863447681] (Long)                 msisdn          'Additional Data (MSISDN)'
INFO:  49: FIXED(  3):n    [978] (Integer)                       currencyCode    'Currency Code, Transaction'
INFO:  90: VAR3 (  4):xn   [0] (Integer)                         originalData    'Original data elements'
```

## Other ISO8583 Solutions ##
[JPos](http://www.jpos.org/)
Full ISO8583 solution, including transport:
> ... "the de-facto OpenSource implementation of the international ISO-8583 standard" (AGPL license/commercial license)

[j8583](http://j8583.sourceforge.net/)
> "j8583 is a Java library to generate and read ISO8583 messages. It does not handle sending or reading them over a network connection, but it does parse the data you have read and can generate the data you need to write, or write it directly to an OutputStream." (GNU Lesser General Public License)

[nucleus8583](https://github.com/robbik/nucleus8583)
> "nucleus8583 is java based, OSGi compliant implementation of the ISO-8583 standard protocol.
> Although nucleus8583 is very small and very simple, nucleus8583 has very extreme
> performance. " (Apache License 2.0)
---
layout: post
tags: 
  - jaxb
title: Friendship between CDATA and JAXB
---

Hello, today we are talking about JAXB. This technology lets us serialize java objects into _xml_. It is pretty old one
and exists for long time since java ee . Its API is simple and easy to use, however there are some nuances as somewhere else.
They occur each time when you want to do something nontrivial. In my case, I have to put some fields into CDATA blocks and
leave their content without any changes. 

There is an example below. It is a class _Person_. Just in case, all examples are written in Kotlin.

```kotlin
@XmlRootElement(name = "person")
@XmlAccessorType(XmlAccessType.FIELD)
class Parent {
    @XmlJavaTypeAdapter(CDATAAdapter::class)
    @XmlElement(name = "name")
    var name: String = ""
    @XmlElementWrapper(name = "children")
    @XmlElements(
            XmlElement(name = "boy", type = Boy::class),
            XmlElement(name = "girl", type = Girl::class))
    var children: Collection<Child> = emptyList()
}

open class Child {
    var name: String = ""
}

class Girl : Child()
class Boy : Child()
```

The field _name_ is need to be put into a _CDATA_ block. In order to do it, we add a `@XmlJavaTypeAdapter` annotation 
and implement own _CDATAAdapter_ interface on ourselves. Have a look at this implementation:

```kotlin
class CDATAAdapter : XmlAdapter<String, String>() {
    override fun unmarshal(value: String?): String {
        return if (value == null || value.isBlank())
            return ""
        else
            value.trim().removePrefix("&lt![CDATA[").removeSuffix("]]&gt")
    }

    override fun marshal(value: String?): String {
        return if (value == null || value.isBlank())
            ""
        else
            "&lt![CDATA[$value]]&gt"
    }
}
```
You might notice that it is not sophisticated at all - just simple manipulations with strings, and will be right. 
The next step is say our serializator or our marshaller don't change strings that are already changed by _CDATAAdapter_.

```kotlin
private val MARSHALLER: Marshaller by lazy {
    val context = JAXBContext.newInstance(Parent::class.java)
    val marshaller = context.createMarshaller()
    marshaller.setProperty(Marshaller.JAXB_ENCODING, StandardCharsets.UTF_8.toString())
    marshaller.setProperty(Marshaller.JAXB_FORMATTED_OUTPUT, true)
    marshaller.setProperty(com.sun.xml.internal.bind.marshaller.CharacterEscapeHandler::class.java.canonicalName,
            CharacterEscapeHandler { chars, start, end, isAttribute, writer ->
                val value = String(chars, start, end).trim()
                if (value.startsWith("&lt![CDATA[") && value.endsWith("]]&gt"))
                    writer.write(chars, start, end)
                else
                    MinimumEscapeHandler.theInstance.escape(chars, start, end, isAttribute, writer)
            })

    marshaller
}
```

If we don't pay attention to some configurations about encoding and "pretty" formatting, we will see a parameter 
for setting a string handler. We use our custom string handler.

What does it do? The logic is simple. It handles strings in two different ways depend upon an input string 
is changed by _CDATAAdapter_ or not. If it is not changed, it delegates the invocation to an internal handler otherwise
it adds _CDATA_ tags to the input string.

That's all and now we are ready to check the work:

```kotlin
val parent = Parent().apply {
        this.name = "John"
        this.children = arrayListOf(
                Girl().apply {
                    this.name = "Margaret"
                },
                Boy().apply {
                    this.name = "Steve"
                }
        )
    }
 
MARSHALLER.marshal(parent, System.out)
```
If we execute the code above, we will see the next output:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<person>
    <name><![CDATA[John]]></name>
    <children>
        <girl>
            <name>Margarette</name>
        </girl>
        <boy>
            <name>Steve</name>
        </boy>
    </children>
</person>
```




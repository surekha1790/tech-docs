### What is SBE:
Simple Binary Encoding used heavily in low latency messaging systems like TREP.
SBE Encoders and Decoders are used to convert data into binary format and publish on to a low latency messaging system then consume and decode the data. It is heavily used in trading platforms and market data feeds for speed and low latency.

### Why SBE:
Traditional serialisation format like XML, JSON are slow for low latency applications. Other formats like ProtBuf, Avro may be efficient but still include overhead or complexity that SBE avoids.

### Benefits of SBE:
Minimise message size to reduce bandwidth.
Maximise encode/decode speed to reduce processing latency.
Provide a fixed  and predictable message layout.
Allows versioning and schema evolution
Allow direct memory access

### How SBE Works:
You define your messages and fields in SBE XML Schema
Generate Java Encoder/Decoders with Codecs Tool (uk.co.real_logic.sbe.SbeTool)
Use these generated codecs in application to encode/decode messages from/to binary buffers
Message are fixed length and contains optional fields as well
It works on buffers so no intermediate object creation which reduces GC overhead.
Send messages over the network or messaging layer like TREP

### What is Schema:
It is a xml format schema to define what kind of fields will be present in the message to encode/decode

### What are the components in Message:
Fields can be primitive data types, enums, repeating groups, composites

### Things to remember while creating SBE message schema

SBE contains multiple parts in the message
![img.png](img.png)
1. Header: It contains mandatory fields like the version of the message. It can also contain more fields when necessary.
2. Root Fields: Static fields of the message. Their block size is predefined and cannot be changed. They can also be defined as optional.
3. Repeating Groups: These represent collection-type presentations. Groups can contain fields and also inner groups to be able to represent more complex structures.
4. Variable Data Fields: These are fields for which we can’t determine their sizes ahead. String and Blob data types are two examples. They’ll be at the end of the message.

### Example SBE Schema:
    <sbe:messageSchema  xmlns:sbe="http://fixprotocol.io/2016/sbe"
                                                 xmlns:xi="http://www.w3.org/2001/XInclude"
                                                 package="com.sbe.codec"
                                                 id="1"
                                                 version="1"
                                                 semanticVersion="0.1"
                                                 description="SBE Codec"
                                                 byteOrder="littleEndian">

                 <types>
                      <type name="price" primitiveType="unit64" semanticType=”Price”/>
                      <type name=qty" primtiveType="unit32" semanticType=”int”/>
                </types>
                <message name="MarketData" id="1" description="Market Data Message">
                      <field name="price" id="44" type="price" semanticType=”Price”/>
                     <field name="quantity" id="38" type="qty" semanticType=”int”/>
                </message>
       </sbe:messageSchema>


#### These can also be separated as
##### some-types.xml → contains all type declarations

    <xml version=”1.0” encoding=”UTF-8”?>
    <types>
        <type name="price" primitiveType="unit64" semanticType=”Price”/>
        <type name=qty" primtiveType="unit32" semanticType=”int”/>
    
        <enum name=”sbeUnit” encodingType=”unit8>
              <validValue name=”DAYS”>0</validValue> 
        </enum>
    
        <composite name=”header”>
             <type name=”version” primitiveType=”unit16”/>
        </composite>
    </types>

##### some-message.xml → contains sbe message schema

        <xml version=”1.0” encoding=”UTF-8”?>
        <message name="MarketData" id="1" description="Market Data Message">
            <field name="price" id="44" type="price" semanticType=”Price”/>
            <field name="quantity" id="38" type="qty" semanticType=”int”/>
            <group name=”parties” id=”342” dimensionType=”<compositeType>”>
                <field name="partyId" id="1245" type="int" semanticType=”int”/>
            </group>

            <data name=”currency” id=”15” type=”String” semanticType=”String”/>
        </message>

##### Now, include all these in schema.xml which will be used to generate codecs
        <sbe:messageSchema  xmlns:sbe="http://fixprotocol.io/2016/sbe"
                            xmlns:xi="http://www.w3.org/2001/XInclude"
                            package="com.sbe.codec"
                            id="1"
                            version="1"
                            semanticVersion="0.1"
                            description="SBE Codec"
                            byteOrder="littleEndian">   
            <xi:include href=”pathToType.xml”/>
            <xi:include href=”pathToMessage.xml”/>
        </sbe:messageSchema>


### How to Generate Codecs:
Use SBETool (uk.co.real_logic.sbe.SbeTool) to generate Java Codecs. Maven plugin configuration can be used to do this task.

#### Example:
        <plugin>
            <groupId> org.codehaus.mojo</groupId>
            <artifactId>exec-maven-plugin</artifactId>
            <version>3.2.0</version>
            <executions>
            <execution>
                <id>sbe</id>
                <phase>generate-sources</phase>
                <goals>
                    <goal>java</goal>
                </goals>
                <configuration combine.self="override">
                    <skip>false</skip>
                    <includeProjectDependencies>false</includeProjectDependencies>
                    <includePluginDependencies>true</includePluginDependencies>
                    <mainClass>uk.co.real_logic.sbe.SbeTool</mainClass>
                    <systemProperties>
                        <systemProperty>
                            <key>sbe.output.dir</key>
                            <value>${project.build.directory}/generated-sources/java</value>
                        </systemProperty>
                        <systemProperty>
                            <key>sbe.xinclude.aware</key>
                            <value>true</value>
                        </systemProperty>
                     </systemProperties>
                    <arguments>
                         <argument>${project.build.resources[0].directory}/schema.xml</argument>
                    </arguments>
                </configuration>
             </execution>
             </executions>
            <dependencies>
            <dependency>
                <groupId>uk.co.real-logic</groupId>
                <artifactId>sbe-tool</artifactId>
                <version>${sbe.version}</version>
            </dependency>
            <dependency>
                <groupId>org.agrona</groupId>
                <artifactId>agrona</artifactId>
                <version>${agrona.version}</version>
            </dependency>
            </dependencies>
        </plugin>

This maven config generates SBE Encoders/Decoders which will be used to encode/decode fix message
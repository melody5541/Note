# Note
   @Test public void
    can_validate_namespace_attributes_when_namespace_aware_is_set_to_false() {
        given().
                config(config().xmlConfig(xmlConfig().namespaceAware(false))).
        when().
                get("/namespace-example2").
        then().
                body("soapenv:Envelope.soapenv:Body.ns1:getBankResponse.@xmlns:ns1", equalTo("http://thomas-bayer.com/blz/"));
    }


     @Test
    public void xmlPathWorksWithSoap() throws Exception {
        // Given
        String soap = "<?xml version=\"1.0\" encoding=\"UTF-8\" standalone=\"yes\"?>\n" +
                "<env:Envelope \n" +
                "    xmlns:soapenc=\"http://schemas.xmlsoap.org/soap/encoding/\" \n" +
                "    xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\" \n" +
                "    xmlns:env=\"http://schemas.xmlsoap.org/soap/envelope/\" \n" +
                "    xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\">\n" +
                "    <env:Header/>\n" +
                "\n" +
                "<env:Body>\n" +
                "    <n1:importProjectResponse \n" +
                "        xmlns:n1=\"n1\" \n" +
                "        xmlns:n2=\"n2\" \n" +
                "        xsi:type=\"n2:ArrayOfProjectImportResultCode\">\n" +
                "        <n2:ProjectImportResultCode>\n" +
                "            <n2:code>1</n2:code>\n" +
                "            <n2:message>Project 'test1' import was successful.</n2:message>\n" +
                "        </n2:ProjectImportResultCode>\n" +
                "    </n1:importProjectResponse>\n" +
                "</env:Body></env:Envelope>";
        // When

        XmlPath xmlPath = new XmlPath(soap);

        // Then
        assertThat(xmlPath.getString("Envelope.Body.importProjectResponse.ProjectImportResultCode.code"), equalTo("1"));
    }


以WeatherWebService的getWeatherbyCityName SOAP1.2为例
需要两个对象：
请求对象（GetWeatherbyCityName）
响应对象（GetWeatherbyCityNameResponse）

Java代码
package jaxb.soap;

import javax.xml.bind.annotation.XmlElement;
import javax.xml.bind.annotation.XmlRootElement;

@XmlRootElement
public class GetWeatherbyCityName {
    @XmlElement
    private String theCityName;

    private GetWeatherbyCityName() {
    }
    public static GetWeatherbyCityName create() {
        return new GetWeatherbyCityName();
    }
    public GetWeatherbyCityName theCityName(String theCityName) {
        this.theCityName = theCityName;
        return this;
    }
}


Java代码  收藏代码
package jaxb.soap;

import java.util.ArrayList;
import java.util.List;

import javax.xml.bind.annotation.XmlElement;
import javax.xml.bind.annotation.XmlElementWrapper;
import javax.xml.bind.annotation.XmlRootElement;

@XmlRootElement
public class GetWeatherbyCityNameResponse {

    @XmlElementWrapper(name = "getWeatherbyCityNameResult")
    @XmlElement(name = "string")
    private List<String> strings = new ArrayList<String>();

    public List<String> getStrings() {
        return strings;
    }

}

package-info.java 不能少，给请求对象GetWeatherbyCityName加上命名空间的，修改前缀
Java代码  收藏代码
@javax.xml.bind.annotation.XmlSchema(
    namespace = "http://WebXml.com.cn/",
    elementFormDefault = javax.xml.bind.annotation.XmlNsForm.QUALIFIED
)
package jaxb.soap;


请求WebService并解析为GetWeatherbyCityNameResponse对象
Java代码  收藏代码
package jaxb;

import java.io.InputStream;
import java.io.OutputStream;
import java.net.HttpURLConnection;
import java.net.URI;
import java.net.URL;

import javax.xml.bind.JAXBContext;
import javax.xml.bind.JAXBElement;
import javax.xml.bind.Marshaller;
import javax.xml.bind.Unmarshaller;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.soap.MessageFactory;
import javax.xml.soap.SOAPBody;
import javax.xml.soap.SOAPConstants;
import javax.xml.soap.SOAPEnvelope;
import javax.xml.soap.SOAPMessage;

import jaxb.soap.GetWeatherbyCityNameResponse;
import jaxb.soap.GetWeatherbyCityName;

import org.w3c.dom.Document;

public class JaxbTest {

    private static String uri = "http://www.webxml.com.cn/WebServices/WeatherWebService.asmx";

    public static void main(String[] args) throws Exception {
        URL url = URI.create(uri).toURL();
        HttpURLConnection connection = (HttpURLConnection) url.openConnection();

        connection.setDoOutput(true);
        connection.setRequestMethod("POST");
        connection.setRequestProperty("Content-Type", "application/soap+xml; charset=utf-8");

        // 发送数据
        OutputStream outputStream = connection.getOutputStream();

        Document requestDocument = DocumentBuilderFactory.newInstance().newDocumentBuilder().newDocument();
        Marshaller marshaller = JAXBContext.newInstance(GetWeatherbyCityName.class).createMarshaller();
        marshaller.marshal(GetWeatherbyCityName.create().theCityName("南京"), requestDocument);
        SOAPMessage requestSOAPMessage = MessageFactory.newInstance(SOAPConstants.SOAP_1_2_PROTOCOL).createMessage();
        SOAPBody soapBody = requestSOAPMessage.getSOAPBody();
        soapBody.addDocument(requestDocument);
        SOAPEnvelope soapEnvelope = requestSOAPMessage.getSOAPPart().getEnvelope();
        soapEnvelope.removeNamespaceDeclaration("env");
        soapEnvelope.addNamespaceDeclaration("soap12", "http://www.w3.org/2003/05/soap-envelope");
        soapEnvelope.addNamespaceDeclaration("xsi", "http://www.w3.org/2001/XMLSchema-instance");
        soapEnvelope.addNamespaceDeclaration("xsd", "http://www.w3.org/2001/XMLSchema");
        soapEnvelope.setPrefix("soap12");
        soapEnvelope.removeChild(soapEnvelope.getHeader());
        soapBody.setPrefix("soap12");
        requestSOAPMessage.writeTo(outputStream);

        // 接收数据
        InputStream inputStream = connection.getInputStream();

        SOAPMessage responseSOAPMessage = MessageFactory.newInstance(SOAPConstants.SOAP_1_2_PROTOCOL).createMessage(null, inputStream);
//      responseSOAPMessage.writeTo(System.out);
        Unmarshaller unmarshaller = JAXBContext.newInstance(GetWeatherbyCityNameResponse.class).createUnmarshaller();
        JAXBElement<GetWeatherbyCityNameResponse> jaxbElement = unmarshaller.unmarshal(responseSOAPMessage.getSOAPBody().extractContentAsDocument(), GetWeatherbyCityNameResponse.class);
        GetWeatherbyCityNameResponse response = jaxbElement.getValue();
        System.out.println(response.getStrings());

        outputStream.close();
        inputStream.close();
        connection.disconnect();
    }

}

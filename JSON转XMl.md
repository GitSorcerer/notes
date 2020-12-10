# JSON转XMl

## POM

```xml
  <dependency>
      <groupId>org.json</groupId>
      <artifactId>json</artifactId>
      <version>20171018</version>
    </dependency>
    <dependency>
      <groupId>dom4j</groupId>
      <artifactId>dom4j</artifactId>
      <version>1.6.1</version>
    </dependency>
    <dependency>
      <groupId>de.odysseus.staxon</groupId>
      <artifactId>staxon</artifactId>
      <version>1.3</version>
    </dependency>
```

## java

```java
package com.zccs.assets_management.utils;

import de.odysseus.staxon.json.JsonXMLConfig;
import de.odysseus.staxon.json.JsonXMLConfigBuilder;
import de.odysseus.staxon.json.JsonXMLInputFactory;
import de.odysseus.staxon.xml.util.PrettyXMLEventWriter;
import org.apache.commons.lang.StringUtils;
import org.dom4j.Document;
import org.dom4j.io.SAXReader;
import org.json.JSONObject;
import org.json.XML;

import javax.xml.stream.XMLEventReader;
import javax.xml.stream.XMLEventWriter;
import javax.xml.stream.XMLOutputFactory;
import java.io.IOException;
import java.io.StringReader;
import java.io.StringWriter;

/**
 * @author gaoh
 * @version 1.0
 * @date 2019/9/3 11:26
 */
public class JsonUtils {

    public static String xmlToJson(String file) throws Exception {
        //使用DOM4j
        SAXReader saxReader = new SAXReader();
        //读取文件
        Document read = saxReader.read(file);
        //使用json的xml转json方法
        JSONObject jsonObject = XML.toJSONObject(read.asXML());
        //设置缩进转为字符串
        System.out.println(jsonObject.toString(3));
        return jsonObject.toString(3);
    }


    /**
     * <p>
     *     json转换成xml
     * </p>
     * @param json  JSON 字符串
     * @param head  XMl头
     * @param root  根节点
     * @return
     */
    public static String jsonToXml(String json, String head, String root) {
        json = "{ " + "\"" + (StringUtils.isNotBlank(root) ? root : "root") + "\"" + " :   " + json + " }";

        //输入流
        StringReader input = new StringReader(json);
        //输出流
        StringWriter output = new StringWriter();
        //构建配置文件
        JsonXMLConfig config = new JsonXMLConfigBuilder().multiplePI(false).repairingNamespaces(false).build();
        try {
            //xml事件读
            //  This is the top level interface for parsing XML Events.  It provides
            //  the ability to peek at the next event and returns configuration
            //  information through the property interface.
            // 这是最解析XML事件最顶层的接口，它提供了查看下一个事件并通过属性界面返回配置信息的功能。
            XMLEventReader reader = new JsonXMLInputFactory(config).createXMLEventReader(input);
            //这是编写XML文档的顶级界面。
            //验证XML的形式不需要此接口的实例。
            XMLEventWriter writer = XMLOutputFactory.newInstance().createXMLEventWriter(output);
            //创建一个实例使用默认的缩进和换行
            writer = new PrettyXMLEventWriter(writer);
            //添加整个流到输出流，调用next方法，知道hasnext返回false
            writer.add(reader);
            reader.close();
            writer.close();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            try {
                output.close();
                input.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        String xml = output.toString();
        //移除头部标签
        if (output.toString().length() >= 38) {
            if (head == null || "".equals(head)) {
              /*  xml = output.toString().substring(39);
                head = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n";
                xml = head + xml;*/
                System.out.println(xml);
            } else {
                xml = output.toString().substring(39);
                xml = head + xml;
                System.out.println(xml);
                return xml;
            }

        }
//        System.out.println(output);

        return xml;
    }
}

```

## Controller

```java
@RequestMapping("/writerXml")
    public void writerXml(HttpServletRequest request, HttpServletResponse response) {
        response.setContentType("utf-8");

        String param = request.getParameter("param");

        String xml = JsonUtils.jsonToXml(param, null, null);


        PrintWriter writer = null;
        try {
            writer = response.getWriter();
            writer.write(xml);
            writer.flush();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (writer != null) {
                writer.close();
            }
        }

    }
```


---
layout: page
title: Custom Policy Development
description: A guide to developing a custom policy component
product: ag
category: learn
sub-nav-class: Custom Development
weight: 10
type: page
nav-title: Custom Policy Development
---

## Custom Policy Development

<h3 style="color: grey;">Table of Contents</h3>
<ol class="table_of_contents">
	<li><a href="#introduction">Overview</a></li>
	<li><a href="#handler-design">Policy Handler Design</a></li>
	<li><a href="#develop">Developing the Policy Handler</a></li>
	<li><a href="#console">Developing the Policy Handler Console Plug-in</a></li>
	<li><a href="#deploy">Building and Deploying the Policy Handler Plug-ins</a></li>
	<li><a href="#testing">Testing the Policy Handler</a></li>
</ol>

### <a name="introduction"></a>Overview

This document contains the information required to:



#### <a name="data"></a>Prerequisites

* This configuration guide assumes that you’ve already installed the platform. If you need help installing the platform, please see the [install guide](http://docs.akana.com/sp/assets/SOA_Software_Platform_Install_Guide_v70.pdf). 
* If writing policy components for the API Gateway, you will have to create and configure a Policy Manager (PM) and Network Directory (ND) container. This is described in the document [Managing a Simple API](simple-api.html#Installing)
* Install and configure the Eclipse IDE as described in the document [Eclipse Workspace Setup](eclipse-setup.html)


### <a name="handler-design"></a>Policy Handler Design

All Policy Handlers utilize the Policy Handler Framework provided by Akana. The Policy Handler Framework is a specialization of the Message Handler Framework that is also provided by Akana.



![Handler UML](images/handler_uml.png "Handler UML")


![Policy UML](images/policy_uml.png "Policy UML")




The Policy Handler framework works on the notion of Handler chains. These chains are associated with two criteria:


The following diagram shows how handler chains are processed for the IN message (such as an HTTP request):

![IN Message Processing](images/in_processing.png "IN Message Processing")

The following diagram shows how handler chains are processed for the OUT message (such as an HTTP response):

![OUT Message Processing](images/out_processing.png "OUT Message Processing")


### <a name="develop"></a>Developing the Policy Handler

The Policy Handler is developed as an OSGi Plug-in. Please refer to the [OSGi Plug-in Development](osgi-plugin-development.html) document which describes how to set up an Eclipse workspace and create a basic plugin project.

**Note:** Sample policy handlers can be found in the /samples directory that is distributed with the product.

A typical policy handler plug-in will also have these additional folders:

* build: contains the build.xml Ant script and related files

####Create the Assertion Model

All policies are defined using an XML schema which forms the basis for the model object generation using JAXB v2. 


	```
<xs:schema targetNamespace="http://soa.com/products/policymanager/examples/policy/complex" elementFormDefault="qualified" attributeFormDefault="unqualified" xmlns:xs="http://www.w3.org/2001/XMLSchema" xmlns:cp="http://soa.com/products/policymanager/examples/policy/complex">
  <xs:element name="Complex">
  	<xs:complexType>
  		<xs:sequence>
  			<xs:element name="HeaderName" type="xs:string"></xs:element>
  			<xs:element name="Optional" type="xs:boolean"></xs:element>
  		</xs:sequence>
  	</xs:complexType>
  </xs:element>
</xs:schema>
	```

Several Spring beans need to be published by the policy handler by editing the /META-INF/sping/*.xml files:

* Handler Factories
* Template
* Assertion Marshaller


	<bean id="complex.wsphandler.factory" class="com.soa.examples.policy.complex.handler.ComplexPolicyHandlerFactory"/>
	
	<osgi:service  ref="complex.wsphandler.factory" interface="com.soa.policy.wspolicy.handler.WSPHandlerFactory">
        <osgi:service-properties>
            <entry key="name" value="com.soa.examples.complex.in.http.wsphandler.factory"/>
            <entry key="scope" value="concrete"/>
            <entry key="binding" value="http"/>
            <entry key="role" value="provider"/>
        </osgi:service-properties>
    </osgi:service>
    
2. Define the Spring bean for the Policy Template that is used to describe aspects of the policy and publish the OSGi service:

	```
	<!-- Complex policy template -->
	<bean id="complex.policy.template" class="com.soa.examples.policy.complex.template.ComplexPolicyTemplate"/>
	
	<!-- publish the complex policy template. An id property needs to be included that matches the template id -->
	<osgi:service ref="complex.policy.template" interface="com.soa.policy.template.OperationalPolicyTemplate">
		<osgi:service-properties>
			<entry key="name" value="com.soa.examples.policy.complex.template"/>
			<entry key="id" value="policy.complexexample"/>
		</osgi:service-properties>
	</osgi:service>
	```
	
 
	```
    <bean id="complex.jaxb.marshaller" class="com.soa.policy.wspolicy.JaxbAssertionMarshaller"  init-method="init">
    	<property name="assertionQNames">
			<list>
				<ref bean="complex.assertion.name"/>
			</list>
		</property>
		<property name="jaxbPaths">
			<list>
				<value>com.soa.examples.policy.complex.assertion.model</value>
			</list>
		</property>
	</bean>
	
	<!-- complex master/parent policy marshaller -->
    <bean id="complex.assertion.marshaller" class="com.soa.examples.policy.complex.assertion.marshaler.ComplexAssertionMarshaller">
        <property name="jaxbMarshaller" ref="complex.jaxb.marshaller"/>
    </bean>
    
    <!-- publish the complex marshaller -->
	<osgi:service ref="complex.assertion.marshaller" interface="com.soa.policy.wspolicy.AssertionMarshaller">
		<osgi:service-properties>
			<entry key="name" value="com.soa.examples.policy.complex.assertion.marshaller"/>
		</osgi:service-properties>	
	</osgi:service>

	
	

	```
	
####Package Descriptions


2. xxx.xxx.akana.policy.xxxx.assertion
4. xxx.xxx.akana.policy.xxxx.assertion.marshaller


	


#####xxx.xxx.akana.policy.xxxx.assertion
	
	An Assertion is the un-marshalled XML policy that was attached to the Service in the PM console. It has getters and setters for the values of the elements that make up the policy directives. 

#####xxx.xxx.akana.policy.xxxx.assertion.marshaller

* **xxxxAssertionMarshaller** - This is the implementation of the `com.soa.policy.wspolicy.AssertionMarshaller` interface.



In the com.soa.examples.policy.handler.complex policy plug-in, the build/build.xml file contains an 'assertions' target that is responsible for the generation of the model objects.
###<a href="#console"></a>Developing the Policy Handler Console Plug-in

Policies are configured in the Policy Manager console via one of two different mechanisms:

1. **XML Policy** - the XML Policy is used when no Console Plug-in can be found for a policy. Users can simply type the XML assertion into a dialog box with the appropriate XML structure (namespace, localname, etc) and it will be passed into the Policy Handler as an assertion. This is the approached leveraged by the 'com.soa.examples.policy.handler.simple' example.
2. **Custom Console Plugin** - a nicer way to interface with users is via a specific UI designed to render the policy. This is the approach used by the 'com.soa.examples.policy.handler.complex' example.

The Policy Handler Console Plug-in is developed as an OSGi Plug-in. Please refer to the [OSGi Plug-in Development](osgi-plugin-development.html) document which describes how to set up an Eclipse workspace and create plug-in projects. Ensure that you have followed the directions for 'Compiling the complex policy handler example'.

A typical policy handler console plug-in will also have these additional folders:

* build: contains the build.xml Ant script and related files
* META-INF/images: contains the images required by the plug-in
* OSGI-INF/l10n: Localization message bundle
* WebContent/xxxpolicy; contains the JSP files
* WebContent/WEB-INF: contains the tag libraries and web.xml file
	 

Two Spring beans need to be published by the policy handler by editing the /META-INF/sping/*.xml files:

* Policy Renderer
* Faces Config


	    	         
	<osgi:service ref="complex.policy.renderer" interface="com.soa.console.policy.renderer.OperationalPolicyRenderer"/>
	    
2. Define the Spring bean for the Faces configuration and publish the OSGi service. This rarely needs any customization but is required for the policy to work correctly:

	```
	<osgi:service interface="com.soa.console.faces.config.FacesConfig">
		<osgi:service-properties>
			<entry key="name" value="com.soa.examples.complex.policy.faces.config"/>
		</osgi:service-properties>	
		<bean class="com.soa.console.faces.myfaces.MyFacesConfigFactory">
			<property name="location" value="classpath:faces/faces-config.xml"/>
		</bean>
	</osgi:service>
	```
		
####Package Descriptions


2. xxx.xxx.akana.console.policy.xxxx.bean


#####xxx.xxx.akana.console.policy.xxxx.bean

* **xxxxPolicyBean** - Extends the `com.digev.ms.console.struts.forms.MSActionForm` abstract class.

####Localization####

The policy.xxxx.name is a special property and should always match the id from the PolicyRenderer - with a '.name' suffix:

```
policy.complexexample.name=Complex Example Policy
com.soa.examples.console.policy.complex.options.label=Options
com.soa.examples.console.policy.complex.headername.label=Header Name
com.soa.examples.console.policy.complex.optional.label=Optional

```

To localize text in the JSP pages, simply use the <workbench:message> tag:

```
<workbench:message key="com.soa.examples.console.policy.complex.headername.label"/>
```


The WebContent/xxxpolicy directory contains the files required to render the user interface. The entry point is defined in the `PolicyRender.getContentLocation(String policyKey)` method. e.g:

```
public class ComplexPolicyRenderer extends OperationalPolicyRendererBase {

	public String getId() {
		return "policy.complexexample";
	}

	public String getContentLocation(String policyKey) throws GException{
		return "/complexpolicy/complex_policy_details.jsp?policyKey="+policyKey;
	}

	public String getContentLocation(HttpServletRequest request)
			throws GException {
		return getContentLocation(getPolicyKey(request));
	}
}
```

In this case, the complex\_policy\_details.jsp JSP page is passed the PolicyBean and renders the policy details. It also contains a link to the page used to modify the policy. In this example:

```

	<td><b><workbench:message key="com.soa.examples.console.policy.complex.options.label"/></b>
	  			&nbsp;&nbsp;|&nbsp;&nbsp;<a  href="javascript:(new createWindow('<%=request.getContextPath()%>/complexpolicy/modify_complex_policy_details.faces?policyKey=<%=policyKey%>', 'ActionWizardWindow', 10, 10, 550, 200, 'no', 'yes', 'no', 'no', 'no').openWindow());"><workbench:message key="com.soa.examples.console.policy.complex.modify.label"/></a>
	</td>
	  	
``` 

When the 'Save' button is clicked, the PolicyBean is called to process the form and save the assertion:

```
	<h:commandButton  id="finish" style="display:none;" action="#{ComplexPolicyBean.modifyAction}"/>
	<f:verbatim><feat:Button label="save" onclick="submitFinish()"/></f:verbatim>
```

###<a href="#deploy"></a>Building and Deploying the Policy Handler Plug-ins

####Build the Policy Handler






The Policy Manager requires both the 'policy handler' and 'policy handler console' bundles, whereas the Network Director only requires the 'policy handler' bundle.
 



	
	```
	./startup.sh <ND name> –debug 7777
	```
	


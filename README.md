## Spring MVC and Hibernate Project
>透過先前學習的Spring MVC與Hibernate結合，建造一個簡易的客戶關係管理系統(ORM)，基本上內容包含添加新客戶、更新、刪除以及列出所有相關資料，也就是基本的CRUD功能。
>
![image](https://user-images.githubusercontent.com/101872264/215779474-d0308427-d5a1-46cc-a029-3113f03a9d29.png)

## spring-mvc-crud-demo-servlet.xml資料庫連接相關配置
**此處為基本掃描包的範圍、添加對註釋驅動的支持，主要就是為了Spring MVC來用的，提供Controller請求轉發，json自動轉換等功能，以及視圖解析的配置**
```
<!-- Add support for component scanning -->
<context:component-scan base-package="com.luv2code.springdemo" />

<!-- Add support for conversion, formatting and validation support -->
<mvc:annotation-driven/>

<!-- Define Spring MVC view resolver -->
<bean
	class="org.springframework.web.servlet.view.InternalResourceViewResolver">
	<property name="prefix" value="/WEB-INF/view/" />
	<property name="suffix" value=".jsp" />
</bean>
```
**此處主要定義數據源相關連線參數，以及連接池的屬性配置，**
```
<!-- Step 1: Define Database DataSource / connection pool -->
<bean id="myDataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource"
      destroy-method="close">
    <property name="driverClass" value="com.mysql.cj.jdbc.Driver" />
    <property name="jdbcUrl" value="jdbc:mysql://localhost:3306/web_customer_tracker?useSSL=false&amp;serverTimezone=UTC" />
    <property name="user" value="hbstudent" />
    <property name="password" value="hbstudent" /> 

    <!-- these are connection pool properties for C3P0 -->
    <property name="minPoolSize" value="5" />
    <property name="maxPoolSize" value="20" />
    <property name="maxIdleTime" value="30000" />
</bean>  
```
**設置Hibernate的SessionFactory，SessionFactory是Hibernate用來與資料庫溝通，因此給定我們第一步時配置的數據源引用，裡面包含所有與資料庫連線相關資訊；另外，配置實體類別所在位置以供系統自動搜尋以及事務管理配置，同時添加resources資源參考路徑以供後續JSP頁面樣式使用**
```
<!-- Step 2: Setup Hibernate session factory -->
<bean id="sessionFactory"
	class="org.springframework.orm.hibernate5.LocalSessionFactoryBean">
	<property name="dataSource" ref="myDataSource" />
	<property name="packagesToScan" value="com.luv2code.springdemo.entity" />
	<property name="hibernateProperties">
	    <props>
		  <prop key="hibernate.dialect">org.hibernate.dialect.MySQLDialect</prop>
		  <prop key="hibernate.show_sql">true</prop>
		</props>
	</property>
</bean>

<!-- Step 3: Setup Hibernate transaction manager -->
<bean id="myTransactionManager"
        class="org.springframework.orm.hibernate5.HibernateTransactionManager">
    <property name="sessionFactory" ref="sessionFactory"/>
</bean>
    
<!-- Step 4: Enable configuration of transactional behavior based on annotations -->
<tx:annotation-driven transaction-manager="myTransactionManager" />

<!-- Add support for reading web resources: css, image, js, etc ... -->
<mvc:resources location="/resources/" mapping="/resources/**"></mvc:resources>
```

## DAO(Data Access Object)
>參考網址：https://openhome.cc/Gossip/SpringGossip/SpringDAO.html
>
>Spring 的DAO框架讓您在進行資料庫存取時，無須接觸到與所使用特定資料庫的技術相關細節，DAO的全名為Data Access Object，在您的應用程式中，需要使用到資料存取時，是透過一個資料存取介面來操作，而實際上進行資料庫存取的物件要實作該介面，並在規範的方法之 中，實作存取時的相關細節。

**首先必要的，創建實體類別，並將相關資料字段映射到資料庫中的相對應數據列**
```
@Entity
@Table(name="customer")
public class Customer {
	
	@Id
	@GeneratedValue(strategy=GenerationType.IDENTITY)
	@Column(name="id")
	private int id;
	
	@Column(name="first_name")
	private String firstName;
	
	@Column(name="last_name")
	private String lastName;
	
	@Column(name="email")
	private String email;
	
	public Customer()
	{
		
	}

	public int getId() {
		return id;
	}

	public void setId(int id) {
		this.id = id;
	}

	public String getFirstName() {
		return firstName;
	}

	public void setFirstName(String firstName) {
		this.firstName = firstName;
	}

	public String getLastName() {
		return lastName;
	}

	public void setLastName(String lastName) {
		this.lastName = lastName;
	}

	public String getEmail() {
		return email;
	}

	public void setEmail(String email) {
		this.email = email;
	}

	@Override
	public String toString() {
		return "Customer [id=" + id + ", firstName=" + firstName + ", lastName=" + lastName + ", email=" + email + "]";
	}
}
```
**@Repository註解的作用**
>參考網址：https://blog.csdn.net/wqh0830/article/details/96109587
>
>該註解的作用不只是將類識別為Bean，同時它還能將所標註的類中拋出的數據訪問異常封裝為Spring 的數據訪問異常類型。Spring本身提供了一個豐富的並且是與具體的數據訪問技術無關的數據訪問異常結構，用於封裝不同的持久層框架拋出的異常，使得異常獨立於底層的框架

**DAO相關方法介面**
```
public interface CustomerDAO {


	public void saveCustomer(Customer theCustomer);

	public Customer getCustomer(int theId);

	public void deleteCustomer(int theId);

	public List<Customer> searchCustomers(String theSearchName);

	public List<Customer> getCustomers(int theSortField);
}
```
**實作DAO方法實現完整程式碼**
```
@Repository
public class CustomerDAOImpl implements CustomerDAO {

	// need to inject the session factory
	// 此處為Bean Id，與XML檔案中的配置名稱相同，Spring會掃描對應Package中是否存在此Bean Id並自動注入，用於與資料庫溝通
	@Autowired
	private SessionFactory sessionFactory;

	@Override
	public void saveCustomer(Customer theCustomer) {
		
		// get current hibernate session
		Session currentSession = sessionFactory.getCurrentSession();
		
		// save/update the customer ... finally LOL
		currentSession.saveOrUpdate(theCustomer);
	}


	@Override
	public Customer getCustomer(int theId) {
		
		// get the current hibernate session
		Session currentSession = sessionFactory.getCurrentSession();
		
		// now retrieve/read from database using the primary key
		Customer theCustomer = currentSession.get(Customer.class, theId);
		
		return theCustomer;
	}


	@Override
	public void deleteCustomer(int theId) {
		
		// get the current hibernate session
		Session currentSession = sessionFactory.getCurrentSession();
		
		// delete object with primary key
		Query theQuery = 
				currentSession.createQuery("delete from Customer where id=:customerId");
		theQuery.setParameter("customerId", theId);
		
		theQuery.executeUpdate();
	}


	@Override
	public List<Customer> searchCustomers(String theSearchName) {

	// get the current hibernate session
	Session currentSession = sessionFactory.getCurrentSession();
		
	Query theQuery = null;
		
	//
	// only search by name if theSearchName is not empty
	//
	if (theSearchName != null && theSearchName.trim().length() > 0) {

		// search for firstName or lastName ... case insensitive
		theQuery =currentSession.createQuery("from Customer where lower(firstName) like :theName or lower(lastName) like :theName", Customer.class);
		theQuery.setParameter("theName", "%" + theSearchName.toLowerCase() + "%");

	}
	else {
		// theSearchName is empty ... so just get all customers
		theQuery =currentSession.createQuery("from Customer order by lastName", Customer.class);			
	}
		
	// execute query and get result list
	List<Customer> customers = theQuery.getResultList();
				
	// return the results		
	return customers;
		
	}


	@Override
	public List<Customer> getCustomers(int theSortField) {
	// get the current hibernate session
	Session currentSession = sessionFactory.getCurrentSession();
						
	// determine sort field
	String theFieldName = null;
				
	switch (theSortField) {
		case SortUtils.FIRST_NAME: 
			theFieldName = "firstName";
			break;
		case SortUtils.LAST_NAME:
			theFieldName = "lastName";
			break;
		case SortUtils.EMAIL:
			theFieldName = "email";
			break;
		default:
			// if nothing matches the default to sort by lastName
			theFieldName = "lastName";
	}
				
	// create a query  
	String queryString = "from Customer order by " + theFieldName;
	Query<Customer> theQuery = 
			currentSession.createQuery(queryString, Customer.class);
				
	// execute query and get result list
	List<Customer> customers = theQuery.getResultList();
						
	// return the results		
	return customers;
	}
}
```
**list-customers.jsp**

**添加JSTL標籤的引用，創建表單顯示頁面，包含添加成員功能、刪除與更新按鈕以及成員資料顯示表格，並引用resources底下CSS樣式為表單進行美化，**
```
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>
<%@ page import="com.luv2code.springdemo.util.SortUtils" %>

<!DOCTYPE html>

<html>

<head>
	<title>List Customer</title>
	
	<!-- reference our style sheet -->
	
	<link type="text/css"
		  rel="stylesheet"
		  href="${pageContext.request.contextPath}/resources/css/style.css" />
</head>

<body>

	<div id="wrapper">
		<div id="header">
			<h2>CRM - Customer Relationship Manager</h2>
		</div>
	</div>
	
	<div id="container">
			
			<div id="content">
			
				<!-- put new button Add Customer -->
				
				<input type="button" value="Add Customer"
					   onclick="window.location.href='showFormForAdd'; return false;"
					   class="add-button"
				/>
				
				<!--  add a search box -->
				<form:form action="search" method="GET">
					Search customer: <input type="text" name="theSearchName" />
				
					<input type="submit" value="Search" class="add-button" />
				</form:form>
			
				<!-- add our html table here -->
				
				<table>
				
					<!-- setup header links for sorting -->

					<!-- construct a sort link for first name -->
					<c:url var="sortLinkFirstName" value="/customer/list">
						<c:param name="sort" value="<%= Integer.toString(SortUtils.FIRST_NAME) %>" />
					</c:url>					

					<!-- construct a sort link for last name -->
					<c:url var="sortLinkLastName" value="/customer/list">
						<c:param name="sort" value="<%= Integer.toString(SortUtils.LAST_NAME) %>" />
					</c:url>					

					<!-- construct a sort link for email -->
					<c:url var="sortLinkEmail" value="/customer/list">
						<c:param name="sort" value="<%= Integer.toString(SortUtils.EMAIL) %>" />
					</c:url>
				
					<tr>
						<th><a href="${sortLinkFirstName}">First Name</a></th>
						<th><a href="${sortLinkLastName}">Last Name</a></th>
						<th><a href="${sortLinkEmail}">Email</a></th>
						<th>Action</th>
					</tr>
					
					<!-- loop over and print our customers -->
					<c:forEach var="tempCustomer" items="${customers}">
					
						
						<!-- construct an "update" link with customer id -->
						<c:url var="updateLink" value="/customer/showFormForUpdate">
							<c:param name="customerId" value="${tempCustomer.id}" />
						</c:url>
						
						<!-- construct an "delete" link with customer id -->
						<c:url var="deleteLink" value="/customer/delete">
							<c:param name="customerId" value="${tempCustomer.id}" />
						</c:url>
						
						<tr>
							<td> ${tempCustomer.firstName} </td>
							<td> ${tempCustomer.lastName} </td>
							<td> ${tempCustomer.email} </td>
							
							<td>
								<!-- display the update link -->
								<a href="${updateLink}">Update</a>
								|
								<a href="${deleteLink}"
								   onclick="if (!(confirm('Are you sure want to delete this customer?'))) return false">Delete</a>
							</td>
						</tr>
					</c:forEach>
				
				</table>
						
			</div>
	</div>
	

</body>

</html>
```

**在web.xml中，可看到以下段落，此處為應用程序添加歡迎頁面，服務器會由上而下的方式尋找這些文件，找尋到的第一個將為用戶使用和呈現**
```
<welcome-file-list>
    <welcome-file>index.jsp</welcome-file>
    <welcome-file>index.html</welcome-file>
</welcome-file-list>
```
**在WebContent底下創建index.jsp文件，並在內容加入以下段落，當服務器啟動，將找尋到此文件，並為用戶自動轉向到指定的路徑下，不會再看到404 Error**
```
<% response.sendRedirect("customer/list"); %>
```
**添加Spring MVC標籤庫參考，創建新增人員的表單頁面，包含基本資料欄位、表單傳送轉向以及添加簡易樣式美化表單**
```
<%@ taglib prefix="form" uri="http://www.springframework.org/tags/form" %>

<!DOCTYPE html>
<html>

<head>

	<title>Save Customer</title>
	
	<link type="text/css"
		  rel="stylesheet"
		  href="${pageContext.request.contextPath}/resources/css/style.css">
		  
	<link type="text/css"
		  rel="stylesheet"
		  href="${pageContext.request.contextPath}/resources/css/add-customer-style.css"> 

</head>

<body>

	<div id="wrapper">
		<div>
			<h2>CRM - Customer Relationship Manager</h2>
		</div>
	</div>
	
	<div id="container">
	
		<h3>Save Customer</h3>
	
		<form:form action="saveCustomer" modelAttribute="customer" method="POST">
		
			<!-- need to associate this data with customer id -->
			<form:hidden path="id" />
		
			<table>
				<tbody>
					<tr>
						<td><label>First name:</label></td>
						<td><form:input path="firstName" /></td>
					</tr>
					
					<tr>
						<td><label>Last name:</label></td>
						<td><form:input path="LastName" /></td>
					</tr>
					
					<tr>
						<td><label>Email:</label></td>
						<td><form:input path="email" /></td>
					</tr>
					
					<tr>
						<td><label></label></td>
						<td><input type="submit" value="Save" class="save" /></td>
					</tr>
				</tbody>
			</table>
		
		</form:form>
		
		<div style="clear; both;"></div>
		
		<p>
			<a href="${pageContext.request.contextPath}/customer/list">Back to List</a>
		</p>
	</div>

</body>

</html>
```
## Service(服務層)
>參考網址：https://blog.csdn.net/Time888/article/details/72811929
>
>參考網址：https://en.wikipedia.org/wiki/Service_layer_pattern
>
>service是業務層，是使用一個或多個模型執行操作的方法。
>* 1. 封裝通用的業務邏輯，操作。如一些數據的檢驗，可以通用處理。
>* 2. 與數據層的交互。
>* 3. 其他請求：如遠程服務獲取數據，如第三方api等
>
>>服務層是一種架構模式，應用在面向服務的 設計範例中，旨在將服務清單中的服務組織到一組邏輯層中。分類到特定層的服務共享功能。這有助於減少與管理服務清單相關的概念開銷，因為屬於同一層的服務處理較少的活動集。將服務分組到功能層可以減少變更的影響。大多數更改僅影響它們所在的層，很少有影響其他層的副作用。這從根本上簡化了服務維護。

**創建Service服務介面，並添加相關方法功能**
```
public interface CustomerService {

	public void saveCustomer(Customer theCustomer);

	public Customer getCustomer(int theId);

	public void deleteCustomer(int theId);

	public List<Customer> searchCustomers(String theSearchName);

	public List<Customer> getCustomers(int theSortField);
}
```
**完成Service介面中方法功能的實現，並透過@Autowired協助注入CustomerDAO對象，調用customerDAO完成各方法功能的實現，另外在此加入@Transactional註解，在方法執行時將自動為我們處理交易的開始與結束減少代碼量**
```
@Service
public class CustomerServiceImpl implements CustomerService {

	// need to inject customer dao
	@Autowired
	private CustomerDAO customerDAO;

	@Override
	@Transactional
	public void saveCustomer(Customer theCustomer) {
		
		customerDAO.saveCustomer(theCustomer);
	}

	@Override
	@Transactional
	public Customer getCustomer(int theId) {
		
		return customerDAO.getCustomer(theId);
	}

	@Override
	@Transactional
	public void deleteCustomer(int theId) {
		
		customerDAO.deleteCustomer(theId);
	}

	@Override
	@Transactional
	public List<Customer> searchCustomers(String theSearchName) {

		return customerDAO.searchCustomers(theSearchName);
	}

	@Override
	@Transactional
	public List<Customer> getCustomers(int theSortField) {
		
		return customerDAO.getCustomers(theSortField);
	}
}
```


## 控制器(Controller)
>參考網址：https://blog.csdn.net/magi1201/article/details/82226289
>
>@GetMapping用於將HTTP get請求映射到特定處理程序的方法註解
具體來說，@GetMapping是一個組合註解，是@RequestMapping(method = RequestMethod.GET)的縮寫。
>
>@PostMapping用於將HTTP post請求映射到特定處理程序的方法註解具體來說，@PostMapping是一個組合註解，是@RequestMapping(method = RequestMethod.POST)的縮寫。
>
>使用GET或是POST取決於方法需求，若是使用GET，瀏覽器會發送一個獲取請求以`獲取資料`；POST則是發佈請求以`傳送資料`。
>
>同時，此處透過@Autowired自動注入CustomerService，透過調用customerService來完成相關方法的使用
```
@Controller
@RequestMapping("/customer")
public class CustomerController {
	
	// need to inject our customer service
	@Autowired
	private CustomerService customerService;

	@GetMapping("/list")
	public String listCustomers(Model theModel, @RequestParam(required=false) String sort) {
		
		// get customers from the service
		List<Customer> theCustomers = null;
				
		// check for sort field
		if (sort != null) {
			int theSortField = Integer.parseInt(sort);
			theCustomers = customerService.getCustomers(theSortField);			
		}
		else {
			// no sort field provided ... default to sorting by last name
			theCustomers = customerService.getCustomers(SortUtils.LAST_NAME);
		}
				
		// add the customers to the model
		theModel.addAttribute("customers", theCustomers);
				
		return "list-customers";
	}
	
	@GetMapping("/showFormForAdd")
	public String showFormForAdd(Model theModel) {
		
		// create model attribute to bind form data
		Customer theCustomer = new Customer();
		
		theModel.addAttribute("customer", theCustomer);
		
		return "customer-form";
	}
	
	@PostMapping("/saveCustomer")
	public String saveCustomer(@ModelAttribute("customer") Customer theCustomer) {
		
		// save the customer using our service
		customerService.saveCustomer(theCustomer);
		
		return "redirect:/customer/list";
	}
	
	@GetMapping("/showFormForUpdate")
	public String showFormForUpdate(@RequestParam("customerId") int theId,
									Model theModel) {
		
		// get the customer from the service
		Customer theCustomer = customerService.getCustomer(theId);
		
		// set customer as a model attribute to pre-populate the form
		theModel.addAttribute("customer", theCustomer);
		
		// send over to our form		
		return "customer-form";
	}
	
	@GetMapping("/delete")
	public String deleteCustomer(@RequestParam("customerId") int theId) {
		
		// delete the customer
		customerService.deleteCustomer(theId);
		
		return "redirect:/customer/list";
	}
	
	@GetMapping("/search")
    public String searchCustomers(@RequestParam("theSearchName") String theSearchName,
                                    Model theModel) {
        // search customers from the service
        List<Customer> theCustomers = customerService.searchCustomers(theSearchName);
                
        // add the customers to the model
        theModel.addAttribute("customers", theCustomers);
        return "list-customers";        
    }
}
```

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
**設置Hibernate的SessionFactory，SessionFactory是Hibernate用來與資料庫溝通，因此給定我們第一步時配置的數據源引用，裡面包含所有與資料庫連線相關資訊；另外，配置實體類別所在位置以供系統自動搜尋以及事務管理配置**
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

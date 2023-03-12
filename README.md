## Spring Boot - Thymeleaf - CRUD Database
>參考網址：https://www.thymeleaf.org/doc/tutorials/3.1/usingthymeleaf.html
>
>Thymeleaf 是一種適用於 Web 和獨立環境的現代服務器端 Java 模板引擎，能夠處理 HTML、XML、JavaScript、CSS 甚至純文本。
>
>* Thymeleaf 在有網絡和無網絡的環境下皆可運行，即它可以讓美工在瀏覽器查看頁面的靜態效果，也可以讓程序員在服務器查看帶數據的動態頁面效果。這是由於它支持html 原型，然後在html 標籤裡增加額外的屬性來達到模板+數據的展示方式。瀏覽器解釋html 時會忽略未定義的標籤屬性，所以thymeleaf 的模板可以靜態地運行；當有數據返回到頁面時，Thymeleaf 標籤會動態地替換掉靜態內容，使頁面動態顯示。
>
>* Thymeleaf 開箱即用的特性。它提供標準和spring標準兩種方言，可以直接套用模板實現JSTL、OGNL表達式效果，避免每天套模板、改jstl、改標籤的困擾。同時開發人員也可以擴展和創建自定義的方言。
>* Thymeleaf 提供spring標準方言和一個與SpringMVC 完美集成的可選模塊，可以快速的實現表單綁定、屬性編輯器、國際化等功能

### 以下演示搭配thymeleaf模板完成顯示頁面，並提供CRUD基本功能
**添加Thymeleaf、JPA、MySQL等依賴項目**
```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
	<groupId>mysql</groupId>
	<artifactId>mysql-connector-java</artifactId>
</dependency>
```

**Employee.java程式碼，創建實體類別，並將相關屬性與資料庫中的字段相互映射**
```
@Entity
@Table(name="employee")
public class Employee {

	// define fields
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
	
	// define constructors
	public Employee() {
		
	}
	
	public Employee(int id, String firstName, String lastName, String email) {
		this.id = id;
		this.firstName = firstName;
		this.lastName = lastName;
		this.email = email;
	}

	public Employee(String firstName, String lastName, String email) {
		this.firstName = firstName;
		this.lastName = lastName;
		this.email = email;
	}
	
	// define getter/setter
	以下省略...
}
```

**application.properties在內容中添加資料庫連接相關資訊**
```
#
# JDBC properties
#
spring.datasource.url=jdbc:mysql://localhost:3306/employee_directory?useSSL=false&serverTimezone=UTC
spring.datasource.username=hbstudent
spring.datasource.password=hbstudent
```

**EmployeeRepository.java程式碼，利用Spring Data JPA完成DAO類別，透過JpaRepository由Spring提供CRUD等功能**
```
public interface EmployeeRepository extends JpaRepository<Employee, Integer> {

	// that's it ... no need to write any code
	
	// add a method to sort by last name
	public List<Employee> findAllByOrderByLastNameAsc();
}
```

**EmployeeService.java程式碼，創建Service方法接口**
```
public interface EmployeeService {

	public List<Employee> findAll();
	
	public Employee findById(int theId);
	
	public void save(Employee theEmployee);
	
	public void daleteById(int theId);
}
```

**EmployeeServiceImpl.java程式碼，完成Service接口方法實現，基本上僅是委託調用DAO方法**
```
@Service
public class EmployeeServiceImpl implements EmployeeService {
	
	private EmployeeRepository employeeRepository;
	
	@Autowired
	public EmployeeServiceImpl(EmployeeRepository theEmployeeRepository) {
		employeeRepository = theEmployeeRepository;
	}

	@Override
	public List<Employee> findAll() {		
		return employeeRepository.findAllByOrderByLastNameAsc();
	}

	@Override
	public Employee findById(int theId) {
		Optional<Employee> result = employeeRepository.findById(theId);
		
		Employee theEmployee = null;
		
		if (result.isPresent()) {
			theEmployee = result.get();
		}
		else {
			// we didn't find the employee
			throw new RuntimeException("Did not find employee id - " + theId);
		}
		return theEmployee;
	}

	@Override
	public void save(Employee theEmployee) {
		employeeRepository.save(theEmployee);
	}

	@Override
	@Transactional
	public void daleteById(int theId) {
		employeeRepository.deleteById(theId);
	}
}
```

**EmployeeController.java程式碼，創建控制器，注入EmployeeService透過Service委託調用DAO方法，並添加方法映射，包含基本CRUD功能**
```
@Controller
@RequestMapping("/employees")
public class EmployeeController {
	
	private EmployeeService employeeService;
	
	public EmployeeController(EmployeeService theEmployeeService) {
		employeeService = theEmployeeService;
	}
	
	// add mapping for "/list"	
	@GetMapping("/list")
	public String listEmployees(Model theModel) {
		
		// get employees from db
		List<Employee> theEmployees = employeeService.findAll();
		
		// add to the spring model
		theModel.addAttribute("employees", theEmployees);
		
		return "employees/list-employees";
	}
	
	@GetMapping("/showFormForAdd")
	public String showFormForAdd(Model theModel) {
		
		// create model attribute to bind form data
		Employee theEmployee = new Employee();
		
		theModel.addAttribute("employee", theEmployee);
		
		return "employees/employee-form";
	}
	
	@GetMapping("/showFormForUpdate")
	public String showFormForUpdate(@RequestParam("employeeId") int theId,
									Model theModel) {
		
		// get the employee from the service
		Employee theEmployee = employeeService.findById(theId);
		
		// set employee as a model attribute to pre-populate the form
		theModel.addAttribute("employee", theEmployee);
		
		// send over to our form
		return "employees/employee-form";
	}
	
	@PostMapping("/save")
	public String saveEmployee(@ModelAttribute("employee") Employee theEmployee) {
		
		// save the employee
		employeeService.save(theEmployee);
		
		// use a redirect to prevent duplicate submissions
		return "redirect:/employees/list";
	}
	
	@GetMapping("/delete")
	public String delete(@RequestParam("employeeId") int theId) {
		
		// delete the employee
		employeeService.daleteById(theId);
		
		// redirect to /employees/list
		return "redirect:/employees/list";
	}
}
```

**於src/main/resources/static/index.html添加以下內容，當沒有給出路徑，將自動轉向到給定映射路徑`employees/list`**
```
<meta http-equiv="refresh"	
	  content="0; URL='employees/list'">
```

**list-employees.html程式碼，引用Bootstrap為表單增添樣式，此為主頁面，列出所有人員數據同時加入新增資料按鈕、更新與刪除功能，在更新與刪除功能中嵌入數據ID，在控制器方法中透過`@RequestParam`獲取指定ID以處理特定人員資料**
```
<!DOCTYPE HTML>
<html lang="en" xmlns:th="http://www.thymeleaf.org">

<head>
	<!-- Required meta tags -->
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    
    <!-- Bootstrap CSS -->
    <link rel="stylesheet"
      href="https://stackpath.bootstrapcdn.com/bootstrap/4.2.1/css/bootstrap.min.css">
  </head>
</head>

<body>

<div class="container">

	<h3>Employee Directory</h3>
	<hr>
	
	<!-- Add a button -->
	<a th:href="@{/employees/showFormForAdd}"
		class="btn btn-primary btn-sm mb-3">
		Add Employee	
	</a>
	<table class="table table-borderd table-striped">
		<thead class="thead-dark">
			<tr>
				<th>FristName</th>
				<th>listName</th>
				<th>Email</th>
				<th>Action</th>
			</tr>
		</thead>
		
		<tbody>
			<tr th:each="tempEmployee : ${employees}">
				<td th:text="${tempEmployee.firstName}" />
				<td th:text="${tempEmployee.lastName}" />
				<td th:text="${tempEmployee.email}" />
				
				<td>
					<!-- Add "update" button/link -->
					<a th:href="@{/employees/showFormForUpdate(employeeId=${tempEmployee.id})}"
					   class="btn btn-info btn-sm">
						Update
					</a>
					
					<!-- Add "delete" button/link -->
					<a th:href="@{/employees/delete(employeeId=${tempEmployee.id})}"
					   class="btn btn-danger btn-sm"
					   onclick="if (!(confirm('Are you sure tou want to delete this employee?'))) return false">
						Delete
					</a>
				</td>
			</tr>
		</tbody>
	
	</table>
	
</div>

</body>

</html>
```

**employee-form.html程式碼，此為新增人員數據表單，在內容中添加各項數據輸入欄位，同樣引用Bootstrap為表單增添樣式，首次加載表單時會調用Getter方法進行預處理；而當提交表單時則調用Setter方法，並執行`@{/employees/save}`映射路徑方法，同時數據綁訂在`${employee}`當中**
```
<!DOCTYPE HTML>
<html lang="en" xmlns:th="http://www.thymeleaf.org">

<head>
	<!-- Required meta tags -->
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1, shrink-to-fit=no">
    
    <!-- Bootstrap CSS -->
    <link rel="stylesheet"
      href="https://stackpath.bootstrapcdn.com/bootstrap/4.2.1/css/bootstrap.min.css"
      integrity="sha384-GJzZqFGwb1QTTN6wy59ffF1BuGJpLSa9DkKMp0DgiMDm4iYMj70gZWKYbI706tWS"
      crossorigin="anonymous">
      
  	<title>Save Employee</title>
</head>

<body>

	<div class="container">
	
	<h3>Employee Directory</h3>
	<hr>
	
	<p class="h4 mb-4">Save Employee</p>
	
	<form action="#" th:action="@{/employees/save}"
					 th:object="${employee}" method="POST">
					 
		<!-- Add hidden form field to handle update -->			 
		<input type="hidden" th:field="*{id}" />
					 
		<input type="text" th:field="*{firstName}"
				class="form-control mb-4 col-4" placeholder="First name">
		
		<input type="text" th:field="*{lastName}"
				class="form-control mb-4 col-4" placeholder="Last name">
				
		<input type="text" th:field="*{Email}"
				class="form-control mb-4 col-4" placeholder="Email">	
				
		<button type="submit" class="btn btn-info col-2">Save</button>	
				
	
	</form>
	
	<hr>
	<a th:href="@{/employees/list}">Back to Employees List</a>
	
	</div>

</body>

</html>
```


![image](https://user-images.githubusercontent.com/101872264/224534933-ee49c508-7837-469d-adbf-b7ab8c95b74b.png)

![image](https://user-images.githubusercontent.com/101872264/224534411-e45402c5-c3ef-453b-953f-0f4bc54fcf9a.png)

## The ugly one
The ugly approach is when the Selenium tests are wrapped with try/catch statements as follows:

```import org.testng.Assert;
import org.testng.annotations.Test;
import PageObjects1.HomePage;
import PageObjects1.ResultsPage;
import framework.Screenshot;
public class TestClass extends TestBase{
 
  private static String URL = “http://www.vpl.ca";
 
  @Test
  public void search_For_Keyword_Works() { 
    try {
       HomePage homePage = openSite();
       ResultsPage resultsPage = homePage.searchFor(“java”);
 
       Assert.assertTrue(resultsPage.header().contains(“java”), 
                         “Results Page header is incorrect!”);
    }
    catch (Exception e) {
       Screenshot screenshot = new Screenshot(driver);
       screenshot.capture(“Search_For_Keyword_Works”);
    }
   }
 
   private HomePage openSite() {
     driver.get(URL);
     
     return new HomePage(driver);
   }
 
}
```

If an exception is generated by the page classes (HomePage and ResultsPage) or the assertion fails, a screenshot is taken in the catch clause.
The screenshots will be stored in the /target/screenshots folder of the Maven project. This folder is created in the set up method of the base test class:

```import java.io.File;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.testng.annotations.AfterMethod;
import org.testng.annotations.BeforeClass;
import org.testng.annotations.BeforeMethod;
public class TestBase {
 
  public WebDriver driver;
  private String chromeDriverPath =   
           “./src/test/resources/chromedriver.exe”;
 
  @BeforeClass
  public void setUp_Class() {
     new File(“./target/screenshots”).mkdir();
  }
 
  @BeforeMethod
  public void setUp() {
    System.setProperty(“webdriver.chrome.driver”, chromeDriverPath);
    this.driver = new ChromeDriver();
  }
 
  @AfterMethod
  public void tearDown() { 
    driver.quit();
  }
 
}
```

The base test class has 2 set up methods:
 - one that uses the BeforeClass annotation and that creates the target/screenshots folder
 - another that uses the BeforeMethod annotation and that creates the driver object
The screenshots are taken and saved by the following class:

```package framework;
import java.io.FileOutputStream;
import java.time.LocalDateTime;
import org.openqa.selenium.OutputType;
import org.openqa.selenium.TakesScreenshot;
import org.openqa.selenium.WebDriver;
public class Screenshot {
 
  private WebDriver driver; 
  private String folderPath = “./target/screenshots/”;
 
  public Screenshot(WebDriver driver)
  { 
    this.driver = driver;
  }
  public void capture(String fileName) 
  { 
    try { 
      String now = LocalDateTime.now().toString();
      now = now.replace(“:”, “_”)
               .replace(“;”, “_”)
               .replace(“.”, “_”);
 
      FileOutputStream file = new FileOutputStream(
                 folderPath + fileName + now + “.png”);
 
      byte[] info = 
      ((TakesScreenshot)driver).getScreenshotAs(OutputType.BYTES);
 
      file.write(info);
      file.close(); 
    } 
    catch (Exception ex) {
      throw new RuntimeException(“cannot create screenshot;”, ex);
    }
 
  }
}
```

The screenshot file name includes the now value for uniqueness.
What is wrong with this way of taking screenshots?
2 things are wrong:
 - A good unit test does not include any Java statements such as IF/ELSE, FOR, WHILE, SWITCH or TRY/CATCH. It only uses page methods for opening the site, navigating to the page under test and making assertions on it.
 - Each test will need the duplicated code for taking the screenshot.
The test class does not look good in this case.
Compare it with the page object classes which are clean and simple.

This is the HomePage class:

```import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
public class HomePage {
 private WebDriver driver;
 private static String TITLE = “Vancouver Public Library |”;
 
 private static By SEARCH_BOX_ID = By.id(“edit-search”);
 private static By SEARCH_BUTTON_ID = By.id(“edit-submit1”);
 
 public HomePage(WebDriver driver) {
   this.driver = driver;
 
   if (!driver.getTitle().equals(TITLE))
     throw new RuntimeException(“Home Page is not displayed!”);
 }
 
 public ResultsPage searchFor(String keyword) {
   WebElement searchBox = driver.findElement(SEARCH_BOX_ID);
   searchBox.sendKeys(keyword);
 
   WebElement searchButton = driver.findElement(SEARCH_BUTTON_ID);
   searchButton.click();
 
   return new ResultsPage(driver);
 }
 
}
```

And this is the ResultsPage class:
```
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
public class ResultsPage {
 
 private WebDriver driver;
 
 private static String URL = “vpl.bibliocommons.com”;
 
 private static By HEADER_XPATH = 
    By.xpath(“//h1[@data-test-id = ‘searchTitle’]”);
 
 public ResultsPage(WebDriver driver) {
 
   this.driver = driver;
 
   if (!driver.getCurrentUrl().contains(URL))
     throw new RuntimeException(“Results Page is not displayed!”);
 
 }
 
 public String header() {
   WebElement header = driver.findElement(HEADER_XPATH);
   return header.getText().toLowerCase();
 }
}
```

## The bad one
If the tests cannot include code for taking the screenshots, maybe we can move this code to the page object classes.
HomePage.java changes as follows:

```
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import framework.Screenshot;
public class HomePage {
  private WebDriver driver;
  private static String TITLE = “Vancouver Public Library |”;
 
  private static By SEARCH_BOX_ID = By.id(“edit-search”);
  private static By SEARCH_BUTTON_ID = By.id(“edit-submit1”);
 
  public HomePage(WebDriver driver) {
    this.driver = driver;
 
    if (!driver.getTitle().equals(TITLE))
      throw new RuntimeException(“Home Page is not displayed!”);
  }
 
  public ResultsPage searchFor(String keyword) {
    try {
     
      WebElement searchBox = driver.findElement(SEARCH_BOX_ID);
      searchBox.sendKeys(keyword);
 
      WebElement searchBtn = driver.findElement(SEARCH_BUTTON_ID);
      searchBtn.click();
 
      return new ResultsPage(driver);
    }
    catch (Exception e) {
      Screenshot screenshot = new Screenshot(driver);
      screenshot.capture(“HomePage_SearchFor_”);
 
      throw new RuntimeException(“cannot search!”);
    }
  }
 
}
```

ResultsPage has similar code:

```
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
import framework.Screenshot;
public class ResultsPage {
 
  private WebDriver driver;
 
  private static String URL = “vpl.bibliocommons.com”;
 
  private static By HEADER_XPATH = 
        By.xpath(“//h1[@data-test-id = ‘searchTitle’]”);
 
  public ResultsPage(WebDriver driver) {
    this.driver = driver;
 
    if (!driver.getCurrentUrl().contains(URL))
      throw new RuntimeException(“Results Page is not displayed!”);
  }
 
  public String header() {
 
    try {
      WebElement header = driver.findElement(HEADER_XPATH);
      return header.getText().toLowerCase();
    }
    catch (Exception e) {
      Screenshot screenshot = new Screenshot(driver);
      screenshot.capture(“ResultsPage_Header_”);
 
      throw new RuntimeException(“cannot get the header!”);
    }
 
 }
}
```

In this case, the code of each page method is wrapped with the TRY/CATCH that takes the screenshot in case of exceptions.

The test looks good, though:
```
import org.testng.Assert;
import org.testng.annotations.Test;
import PageObjects2.HomePage;
import PageObjects2.ResultsPage;
public class TestClass extends TestBase{
 
  private static String URL = “http://www.vpl.ca";
 
  @Test
  public void search_For_Keyword_Works() { 
    HomePage homePage = openSite();
    ResultsPage resultsPage = homePage.searchFor(“java”);
 
    Assert.assertTrue(resultsPage.header().contains(“java”), 
                      “Results Page header is incorrect!”);
  }
 
  private HomePage openSite() {
    driver.get(URL);
    return new HomePage(driver);
  }
 
}

```


Why is this bad?
The code duplication moved from the unit tests to the page methods.

Each page method needs to have its code wrapped with the TRY/CATCH to get the screenshots in case of exceptions.

OK, so the screenshots cannot be taken in the tests and they cannot be taken in the page methods. Where will they be taken then?

## The good one
A TestNG listener is the correct solution.
A listener listens to the tests and executes custom code for each test outcome:

```
import java.lang.reflect.Field;
import org.testng.ITestContext;
import org.testng.ITestListener;
import org.testng.ITestResult;
import framework.Screenshot;
import org.openqa.selenium.WebDriver;
/*
   the test listener captures in the OnTestFailure() method the    
   screenshot of the current page both 
   — when an exception happens in a test script
   — when a test script fails because of an assertion
   none of the other listener methods are implemented.
 */
public class TestListener implements ITestListener {
 
  @Override
  public void onTestStart(ITestResult result) { 
  }
 
  @Override
  public void onTestSuccess(ITestResult result) { 
  }
  /*
   the driver parameter of Screenshot constructor is provided by the 
   driver() method;
 
   testResult.getName() returns the name of the test script being 
   executed;
 
   we pass this as parameter to capture() so that the screenshot is 
   named after the test script;
  */
  @Override
  public void onTestFailure(ITestResult testResult) { 
    try {
      Screenshot screenshot = new Screenshot(driver(testResult));
      screenshot.capture(“listener_” + testResult.getName());
    } 
    catch (IllegalAccessException e) {
      e.printStackTrace();
    }  
  }
  @SuppressWarnings(“unchecked”)
  private WebDriver driver(ITestResult testResult) 
     throws IllegalAccessException 
  {
  
    try {
 
      /*
       we use reflection and generics to find the driver object:
 
       1. testResult.getInstance() 
          returns the running test class object
 
       2. testResult.getInstance().getClass() 
          returns the class of testResult (TestClass)
 
       3. testClass.getSuperclass() 
          returns the parent of the test class (BaseTestClass)
 
       4. Field driverField = 
                baseTestClass.getDeclaredField(“driver”)
          returns the driver field of the BaseTestClass
   
       5. driver = 
           (WebDriver)driverField.get(testResult.getInstance());
          gets the value of the driver field from the BaseTestClass
      */
      Class<?extends ITestResult> testClass = 
        (Class<? extends ITestResult>) 
           testResult.getInstance().getClass();
      Class<?extends ITestResult> baseTestClass = 
        (Class<? extends ITestResult>) 
           testClass.getSuperclass();
      Field driverField = baseTestClass.getDeclaredField(“driver”);
      WebDriver driver =   
        (WebDriver)driverField.get(testResult.getInstance()); 
  
      return driver;
    }  
    catch (SecurityException | 
           NoSuchFieldException | 
           IllegalArgumentException e) { 
      throw new RuntimeException("could not get the driver");
    }
 
  }
  @Override
  public void onTestSkipped(ITestResult result) {
  }
  @Override
  public void onTestFailedButWithinSuccessPercentage(ITestResult   
  result) {
  }
  @Override
  public void onStart(ITestContext context) {
  }
  @Override
  public void onFinish(ITestContext context) {
  }
}
```

The code should be fairly easy to understand.

Each listener method gets a parameter with the ITestResult type which is the instance of the test class.

Using this parameter, we can get the driver object from the base test class using reflection.

Once the driver object is available, we use it for taking the screenshot which is named after the running unit test.

How do we use the listener?

No code changes are needed in the page classes. They do not include any code for taking screenshots.

```
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
public class HomePage {
  private WebDriver driver;
  private static String TITLE = “Vancouver Public Library |”;
 
  private static By SEARCH_BOX_ID = By.id(“edit-search”);
  private static By SEARCH_BUTTON_ID = By.id(“edit-submit1”);
 
  public HomePage(WebDriver driver) {
    this.driver = driver;
 
    if (!driver.getTitle().equals(TITLE))
      throw new RuntimeException(“Home Page is not displayed!”);
  }
 
  public ResultsPage searchFor(String keyword) {
    WebElement searchBox = driver.findElement(SEARCH_BOX_ID);
    searchBox.sendKeys(keyword);
 
    WebElement searchButton = driver.findElement(SEARCH_BUTTON_ID);
    searchButton.click();
 
    return new ResultsPage(driver);
  }
 
}
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.WebElement;
public class ResultsPage {
 
  private WebDriver driver;
 
  private static String URL = “vpl.bibliocommons.com”;
 
  private static By HEADER_XPATH = 
        By.xpath(“//h1[@data-test-id = ‘searchTitle’]”);
 
  public ResultsPage(WebDriver driver) {
    this.driver = driver;
 
    if (!driver.getCurrentUrl().contains(URL))
      throw new RuntimeException(“Results Page is not displayed!”);
  }
 
  public String header() {
 
    WebElement header = driver.findElement(HEADER_XPATH);
    return header.getText().toLowerCase();
 
  }
}
```

The test class does not have any code for taking screenshots either.

It uses however the @Listeners annotation with the name of the listener class:
```import org.testng.Assert;
import org.testng.annotations.Listeners;
import org.testng.annotations.Test;
import PageObjects3.HomePage;
import PageObjects3.ResultsPage;
@Listeners(TestListener.class)
public class TestClass extends TestBase{
 
  private static String URL = “http://www.vpl.ca";
 
  @Test
  public void search_For_Keyword_Works() {
 
    HomePage homePage = openSite();
 
    ResultsPage resultsPage = homePage.searchFor(“java”);
 
    Assert.assertTrue(resultsPage.header().contains(“java”), 
                      “Results Page header is incorrect!”);
  }
 
  private HomePage openSite() {
    driver.get(URL);
    return new HomePage(driver);
  }
 
 
}

```

What happens in this case?

The unit test runs. If it generates an exception or the assertion fails, the listener will notice this and take the screenshot of the current page in the OnTestFailure() method.

It should be obvious why this is the correct solution:

The unit test does not include any code for taking screenshots

The page object classes do not include any code for taking screenshots

The screenshots are taken in a single place

If a test class needs its unit tests to take screenshot, it just requires the @Listeners annotation followed by the listener class name
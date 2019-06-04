# 设计模式之组合模式
## What:
>将对象组合成树形结构以表示“部分整体”的层次结构。组合模式使得用户对单个对象和组合对象的使用具有一致性。

组合模式又称为部分整体模式，该模式将对象组合成树形结构以表示"部分-整体"的层次结构。

## Why:
优点：
1、高层模块调用简单；
2、节点自由增加。

缺点：
1.在使用组合模式时，其叶子和树枝的声明都是实现类，而不是接口，违反了依赖倒置原则；
2.要求较高的抽象性，如果节点和叶子有很多差异性的话，比如很多方法和属性都不一样，难以实现组合模式。
## Where:
1.在具有整体和部分的层次结构中，希望通过一种方式忽略整体与部分的差异，客户端可以一致地对待它们。
2.在一个系统中能够分离出叶子对象和容器对象，而且它们的类型不固定，需要增加一些新的类型。
3.需要遍历组织机构，或者处理的对象具有树形结构时使用。如树形菜单，文件、文件夹的管理。

## How:

组合模式有以下几个概念：

**Component**：它是一个抽象角色，为要组合的对象提供统一的接口。

**Leaf**：在组合中表示子节点对象，叶子节点不能有子节点。

**Composite**：定义有枝节点的行为，用来存储部件，实现在Component接口中的有关操作。

举个栗子：一间公司除了有多个部门，还有多个子公司，现在需要遍历公司名字和公司部门名字，这个结构符合树形结构，所以可以使用组合模式。
![Pattern-Composite](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E7%BB%84%E5%90%88%E6%A8%A1%E5%BC%8F/Pattern-Composite.png)

为使程序简单，代码中只有添加、获取分公司名字和部门名字的方法。


Component类：
```java
public abstract class Component {
	
    //部门名字的集合
    public List<String> departmentNameList = new ArrayList();

    //添加

    void addCompanyName(String companyName) { throw new UnsupportedOperationException("不支持添加操作"); }

    void addDepartmentName(String departmentName) { throw new UnsupportedOperationException("不支持添加部门操作"); }

    void addSubCompany(Component company) { throw new UnsupportedOperationException("不支持添加操作"); }

    void addDepartment(Component department) { throw new UnsupportedOperationException("不支持添加部门操作"); }


    //获取

    String getCompanyName() { throw new UnsupportedOperationException("不支持获取操作"); }

    List<String> getAllDepartmentName() { throw new UnsupportedOperationException("不支持获取部门操作"); }

    Component getDepartment() { throw new UnsupportedOperationException("不支持获取部门操作"); }

    void printCompany() { throw new UnsupportedOperationException("不支持获取操作"); }
}
```

Department：
```java
public class Department extends Component {

    //添加部门名字
    @Override
    void addDepartmentName(String departmentName) { departmentNameList.add(departmentName); }

    //获取全部部门名字
    @Override
    List<String> getAllDepartmentName() { return departmentNameList; }
}
```
Company:
```java
public class Company extends Component {

    private String companyName;

    private List<Component> companyList = new ArrayList();

    private Department department = new Department();

    private HashMap<Company, List<Component>> subCompanyMap = new HashMap();//子公司集合


    //添加公司名字
    @Override
    void addCompanyName(String companyName) { this.companyName = companyName; }


    //添加部门对象
    @Override
    void addDepartment(Component department) {
        List<String> departmentNameList = department.getAllDepartmentName();
        Component dept = this.getDepartment();
        for (int i = 0; i < departmentNameList.size(); i++) {
            dept.departmentNameList.add(departmentNameList.get(i));
        }
    }

    //添加子公司对象
    @Override
    void addSubCompany(Component company) {
        companyList.add(company);
        subCompanyMap.put(this, companyList);
    }

    //获取公司名字
    @Override
    String getCompanyName() { return this.companyName; }

    //获取公司部门
    @Override
    Component getDepartment() { return department; }

    //打印公司结构
    @Override
    void printCompany() {
        Set<Company> set = subCompanyMap.keySet();
        Iterator it = set.iterator();
        while (it.hasNext()) {
            Company company = (Company) it.next();
            System.out.println(company.companyName);//输出总公司名字
            List<String> departmentList = company.getDepartment().departmentNameList;
            //第二层：显示总公司部门
            for (int i = 0; i < departmentList.size(); i++) {
                String departmentName = departmentList.get(i);
                System.out.println("    " + departmentName);
            }
            //第二层：显示分公司名字，第三层：显示子公司部门
            List<Component> subCompanyList = subCompanyMap.get(company);
            for (int i = 0; i < subCompanyList.size(); i++) {
                Component subCompany = subCompanyList.get(i);
                System.out.println("    " + subCompany.getCompanyName());
                List<String> subDepartmentList = subCompany.getDepartment().departmentNameList;
                for (int j = 0; j < subDepartmentList.size(); j++) {
                    System.out.println("        " + subDepartmentList.get(j));
                }
            }
        }
    }
}
```

Test:测试类
```java
public class Test {
    public static void main(String[] args) {
        //共同拥有的部门
        Component department = new Department();
        department.addDepartmentName("人事部");
        department.addDepartmentName("财务部");

        Component companyA = new Company();
        companyA.addCompanyName("北京分公司");
        companyA.getDepartment().addDepartmentName("销售部");
        companyA.addDepartment(department);

        Component companyB = new Company();
        companyB.addCompanyName("广州分公司");
        companyB.getDepartment().addDepartmentName("IT部");
        companyB.getDepartment().addDepartmentName("技术部");
        companyB.addDepartment(department);

        Component companyC = new Company();
        companyC.addCompanyName("香港总公司");
        companyC.getDepartment().addDepartmentName("物流部");
        companyC.addDepartment(department);
        
        //添加子公司
        companyC.addSubCompany(companyA);
        companyC.addSubCompany(companyB);

        //打印公司结构(展示公司名和部门名)
        companyC.printCompany();
    }
}
```
输出结果：
```java
香港总公司
    物流部
    人事部
    财务部
    北京分公司
        销售部
        人事部
        财务部
    广州分公司
        IT部
        技术部
        人事部
        财务部
```
Demo的UNL图：
![Component-UML](https://raw.githubusercontent.com/MuggleLee/PicGo/master/%E8%AE%BE%E8%AE%A1%E6%A8%A1%E5%BC%8F/%E7%BB%84%E5%90%88%E6%A8%A1%E5%BC%8F/Component-UML.jpg)

# 总结
分公司既是“整体”，又是“部分”。作为“整体”，分公司可以任意添加部门；作为“部分”，分公司可以被总公司添加。结合组合模式客户端可以方便的添加子节点，简单的调用方法就可以解决复杂的组合对象。

参考资料：

[https://www.cnblogs.com/lfxiao/p/6816026.html](https://www.cnblogs.com/lfxiao/p/6816026.html)
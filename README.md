## Infra CVE Manage Server

### 服务架构图
<img src="https://raw.githubusercontent.com/nicliuqi/icms/main/imgs/framework.png" alt="服务架构图" />

### 服务组件
- OPS
	
	OPS是Infra自建的一个运维平台，也是基础设施运维的一个数据中心。其为服务提供项目查询接口。
- trivy
	
	[trivy](https://trivy.dev/)是一个全面的多功能扫描器，trivy拥有查找安全问题的扫描器，并定位可以找到这些问题的位置。trivy可扫描的目标包括容器镜像、文件系统、Git远程库、k8s、AWS等。本服务中使用了trivy扫描Git远程库的能力。
  
	为保证服务顺利运行，需要安装trivy，安装步骤如下：
	- 下载trivy的压缩包
		
		执行 `wget https://github.com/aquasecurity/trivy/releases/download/v0.38.3/trivy_0.38.3_Linux-64bit.tar.gz`
	- 解压压缩包
		
		执行 `tar -zxf trivy_0.38.3_Linux-64bit.tar.gz` ，解压后执行 `mv ./trivy /usr/bin` 

- CVE Manager
	
	CVE Manager集多个开源社区的第三方组件为一体，对接中科院的漏洞感知工具vitopia，并对接收到的CVE漏洞做匹配、分发以及进度跟踪。

### 服务流程
1. 从OPS获取项目仓库信息
	
	通过OPS提供的接口（暂无）获取基础设施维护的所有工程的仓库链接。
2. 使用trivy获取各个仓库的packages
	
	通过命令 `trivy repo project_address --list-all-pkgs --format json --output project_name.json` 可以将单个仓库的扫描信息以json文件导出。以项目 https://github.com/opensourceways/issue_pr_board 为例，下面展示了issue_pr_board.json的片段（完整文件可点击[完整文件跳转链接]()查看）。

	```json
	{
	  "SchmeVersion": 2,
	  "ArtifactName": "github.com/opensourceways/issue_pr_board",
	  "ArtifactType": "repository",
	  "Metadata": {
		"ImageConfig": {
		  "architecture": "",
		  "created": "0001-01-01T00:00:00Z",
		  "os": "",
		  "rootfs": {
		    "type": "",
		    "diff_ids": null
			}
		},
		  "config": {}
	  },
	  "Results": [
		{
		  "Target": "go.mid",
		  "Class": "lang-pkgs",
		  "Type": "gomod",
		  "Packages": [
		    {
			  "ID": "github.com/astaxie/beego@v1.12.3",
			  "Name": "github.com/astaxie/beego",
			  "Version": "1.12.3",
			  "Licenses": [
			    "Apache-2.0"
			  ],
			  "DependsOn": [
			    "golang.org/x/crypto@v0.0.0-20200622213623-75b288015ac9",
				"github.com/hashicorp/golang-lru@v0.5.4",
				"github.com/go-sql-driver/mysql@v1.5.0",
				"github.com/pkg/errors@v0.9.1",
				"github.com/shiena/ansicolor@v0.0.0-20151119151921-a422bbe96644",
				"github.com/prometheus/client_golang@v1.7.0",
				"gopkg.in/yaml.v2@v2.2.8"
			  ],
			  "Layer": {}
			},
			{
			  "ID": "github.com/beorn7/perks@v1.0.1",
			  "Name": "github.com/beorn7/perks",
			  "Version": "1.0.1",
			  "Licenses": [
			    "MIT"
			  ],
			  "Indirect": true,
			  "Layer": {}
			},
			{
			  "ID": "github.com/cespare/xxhash/v2@v2.1.1",
			  "Name": "github.com/cespare/xxhash/v2",
			  "Version": "2.1.1",
			  "Licenses": [
				"MIT"
			  ],
			  "Layer": {}
			}
	  ]
	}		
	```
	在trivy生成的json文件中，Results的Packages列出了仓库中所有的依赖文件以及依赖关系。在Results中，Vulnerabilities则是trivy扫描出的CVE漏洞，不过由于无法验证完整性和准确性，暂不使用Vulnerabilities提供的数据。

3. 数据整理
	
	在trivy扫描完所有服务提供的项目仓库后，将各个仓库的第三方组件数据整合，生成projects.yaml和packages_list.yaml两个文件待用。
	- projects.yaml
		
		将各个第三方组件所在的项目地址以列表的方式列出，方便后续CVE的匹配与通知。预期的projects.yaml如下
		```yaml
		github.com/astaxie/beego@v1.12.3: 
		- https://github.com/opensourceways/issue_pr_board
		github.com/cespare/xxhash/v2@v2.1.1:
		- https://github.com/opensourceways/issue_pr_board
		- other project address
		APScheduler@3.6.3:
		- https://github.com/opensourceways/app-meeting-server
		- other project address

		...
	- packages_list.yaml
		
		第三方组件列表的格式可参考[openGauss第三方开源软件列表](https://gitee.com/opengauss/openGauss-third_party/blob/master/Third_Party_Open_Source_Software_List.yaml)。保留有效字段软件包名和版本号，预期的packages_list.yaml如下
		```yaml
		jemalloc:
			version: 5.2.1
		libedit:
			version: 20210910-3.1
		OpenSSL:
			version: 1.1.1n
			
		...
		```
		
		**ATTENTION**: 注意，此处生成的packages_list.yaml需要存放到一个可正常访问的公仓中，方便CVE Manager获取。

4. 接收并处理CVE漏洞信息
	
	提供接口接收从CVE Manager推送的CVE漏洞信息，并通过projects.yaml进行CVE漏洞信息的匹配与通知。通知方式推荐使用向目标仓库提交issue的方式。


# NicooNetwork

[![CI Status](https://img.shields.io/travis/504672006@qq.com/NicooNetwork.svg?style=flat)](https://travis-ci.org/504672006@qq.com/NicooNetwork)
[![Version](https://img.shields.io/cocoapods/v/NicooNetwork.svg?style=flat)](https://cocoapods.org/pods/NicooNetwork)
[![License](https://img.shields.io/cocoapods/l/NicooNetwork.svg?style=flat)](https://cocoapods.org/pods/NicooNetwork)
[![Platform](https://img.shields.io/cocoapods/p/NicooNetwork.svg?style=flat)](https://cocoapods.org/pods/NicooNetwork)

## Example
#   NicooNetworkDemo

  https://github.com/yangxina/NicooNetworkDemo 
  
  OC版： https://github.com/yangxina/SAGANNetWorking
 
  首先将Demo下载到本地，运行，获取到网络数据。
  NicooNetwork 是swift4.2 基于 Alamofire 封装的网络请求组件 。
  组件Demo ： DemoNicooNetwork_Example 。 里外还有单独的Demo。
  非组件化的： NicooNetworkDemo。 
  运行Demo可以看到从网络请求到数据展示在页面上。  
  
1. 封装思路

  使用Alamofire 直接发起请求会调用Alamofire.swift 中的方法： 
     
     // MARK: - Data Request
      /// Creates a `DataRequest` using the default `SessionManager` to retrieve the contents of the specified `url`,
     /// `method`, `parameters`, `encoding` and `headers`.
     ///
     /// - parameter url:        The URL.
     /// - parameter method:     The HTTP method. `.get` by default.
     /// - parameter parameters: The parameters. `nil` by default.
     /// - parameter encoding:   The parameter encoding. `URLEncoding.default` by default.
     /// - parameter headers:    The HTTP headers. `nil` by default.
     ///
     /// - returns: The created `DataRequest`.
  
    public func request(
       _ url: URLConvertible,
       method: HTTPMethod = .get,
       parameters: Parameters? = nil,
       encoding: ParameterEncoding = URLEncoding.default,
       headers: HTTPHeaders? = nil)
       -> DataRequest
    {
        return SessionManager.default.request(
            url,
            method: method,
            parameters: parameters,
            encoding: encoding,
             headers: headers
        )
     }

这个方法是Alamofire暴露给外部调用发起请求的方法。 我们可以看到方法中返回了：
      
      SessionManager.default.request(
        url,
        method: method,
        parameters: parameters,
        encoding: encoding,
        headers: headers
      )
      
  由此我们知道，其实发送网络请求的是这个    SessionManager.default  对象。
  NicooNetwork 组件也是基于Alamofire的SessionManager 来进行封装的。
  由于每个人对于网络请求的封装习惯不一样，所以这里不多阐述封装过程。
  
 2. 怎么使用： 
    
     NicooNetwork是面向协议编程的组件。 要发送一个网络请求之前，我们要做： 
        
        1.封装Api: 我们需要创建一个整个APP网络请求的baseAPI类， 用来处理一些公共的东西。
          demo中叫：YTSGBaseAPI.swift
          
        2.创建一个Service,继承于NicooService， 重写： urlGeneratingRule 方法，将服务URL拼接，
          实现协议： NicooServiceProtocol 
          配置你需要的信息，例如： httpToken, accessToken , header ,version 等等信息。
          并且提供拦截器集中处理 Service错误问题。 demo中叫： YTSGService.swift
          
    准备工作做好了，开始发请求，调用一个APi需要做4件事：
    
        1.创建API类： 新建要请求的API,继承于整个APP网络请求的baseAPI类： YTSGBaseAPI，如：demo中的
        BooksHotListAPI。
        在api中指定API需要的参数名：  
        static let kPageNumber = "pageNumber" 
        static let kPageCount = "pageCount"
        static let kCategoryId = "categoryId" // 可选参数，有分类的时候再传 
        重写 loadData()  方法，添加自己的逻辑
        重写：override func methodName() -> String {} 拼接Url
        重写：override func shouldCache() -> Bool {} 是否需要缓存请求的数据
        如果有固定参数， 可以重写： 
        func reform(_ params: [String: Any]?) -> [String: Any]? {}  直接将固定参数
        定义在API中; 
        需要做分页的： 
        func loadNextPage() -> Int {
            if self.isLoading {
               return 0
            }
            return super.loadData()
        }
        
        override func manager(_ manager: NicooBaseAPIManager, beforePerformSuccess response: NicooURLResponse) -> Bool {
               self.pageNumber += 1
               return true
        }
        外部调用 loadNextPage()
        
        2. 在发起请求的类中: 创建BooksHotListAPI对象： 
        lazy private var hotBooksAPI: BooksHotListAPI = {
             let api = BooksHotListAPI()
             api.paramSource = self
             api.delegate = self
             return api
        }()
        
        实现3个代理方法： 
        // MARK: - NicooAPIManagerParamSourceDelegate,NicooAPIManagerCallbackDelegate
        extension ViewController: NicooAPIManagerParamSourceDelegate, NicooAPIManagerCallbackDelegate {
        
        /// 网络请求参数添加  （这里是追加动态参数， 固定参数可以直接在BooksHotListAPI中拼接）
        ///
        /// - Parameter manager: NicooBaseAPIManager
        /// - Returns: params: [String: Any]
        
        func paramsForAPI(_ manager: NicooBaseAPIManager) -> [String : Any]? {
             MBProgressHUD.showAdded(to: view, animated: false)
              return nil
        }
        
        /// 请求成功回调
        ///
        /// - Parameter manager: NicooBaseAPIManager
        
        func managerCallAPISuccess(_ manager: NicooBaseAPIManager) {
            MBProgressHUD.hideAllHUDs(for: view, animated: false)
            let list = manager.fetchJSONData(BookReformer()) as? BookList
            if manager == hotBooksAPI {
                self.requestSuccess(list)
            }
        }
        
        /// 请求失败回调
        ///
        /// - Parameter manager: manager descriptionNicooBaseAPIManager
        func managerCallAPIFailed(_ manager: NicooBaseAPIManager) {
            MBProgressHUD.hideAllHUDs(for: view, animated: false)
            if manager == hotBooksAPI {
                self.requestFail(error: manager.errorMessage, manager: manager)
            }
        }
       
        }
        
        3.创建解析器： BookReformer
         实现协议： 
         func manager(_ manager: NicooBaseAPIManager, reformData jsonData: Data?) -> Any? {
         if manager is BooksHotListAPI {
         return reformHomePageDatas(jsonData)
         }
         return nil
         }
         将数据解析成BookList的Model对象。
        4.创建Model. 
        解析方式可以自行选择。 Demo中使用苹果自带的解析： Codable
        
       另外还有一个服务的配置： 
       在AppDelegate中添加服务：
          
         func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: 
           [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
                // service 配置
                NicooServiceFactory.shareInstance.dataSource = self
        
       
                return true
          }
          
       实现代理：
   
         // MARK: - NicooServiceFactoryProtocol
        
  
         extension AppDelegate: NicooServiceFactoryProtocol {
    
             /// 自定义的服务
              ///
              /// - Returns: 自定义的服务名
           func servicesKindsOfServiceFactory() -> [String : String] {
                return [ConstValue.kYTSGService: "YTSGService"]
           }
    
              /// 自定义的服务所在的命名空间 （如果自定义的服务是组件，命名空间就为 服务组件名）
              ///
              /// - Parameter service: 服务名
              /// - Returns: m命名空间
          func namespaceForService(_ service: String) -> String? {
              switch service {
                   case ConstValue.kYTSGService:
                   return "NicooNetwork_Example"   // 这里可能会由于NicooNetwork-Example 中的 特殊符号导致找不到Servicee的命名空间
                 default:
                   return nil
          }
        }
  
        }
       
    
         
         
  
 
 

## Requirements

## Installation

NicooNetwork is available through [CocoaPods](https://cocoapods.org). To install
it, simply add the following line to your Podfile:

```ruby
pod 'NicooNetwork'
```

## Author

504672006@qq.com, yangxin@tzpt.com

## License

NicooNetwork is available under the MIT license. See the LICENSE file for more info.

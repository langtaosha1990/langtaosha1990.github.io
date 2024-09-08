---
title: "Data、Dictionary和Model之间的转换"
header:
  #   image: /assets/images/bio-photo.jpg
  overlay_image: /assets/images/bio-photo.jpg
# og_image: /assets/images/bio-photo.png
# caption: "Photo credit: [**Unsplash**](https://unsplash.com)"
last_modified_at: 2024-08-26 10:00:00 +0800
---

<!--  -->

```
import Foundation
import UIKit

let jsonData : Data = """
{
"full_name": "John",
"age": 30,
"isDeveloper": true
}
""".data(using: .utf8)!

// json 转 json 字符串/字典
do {
let jsonObject = try JSONSerialization.jsonObject(with: jsonData, options: [])
if let jsonDictionary = jsonObject as? [String: Any] {
// 处理解析后的字典
print("Name: \(jsonDictionary["name"] ?? "")")
print("Age: \(jsonDictionary["age"] ?? "")")
print("Is Developer: \(jsonDictionary["isDeveloper"] ?? "")")
}
}catch {
print("JSON 解析失败: \(error.localizedDescription)")
}

// json 直接转成 map
do {
if let jsonDictionary = try JSONSerialization.jsonObject(with: jsonData, options: []) as? [String : Any] {
print(jsonDictionary)
} else {
print("不是有效的 JSON 字典")
}
} catch {
print("JSON 解析失败: \(error.localizedDescription)")
}

// 定义 User 结构体并使其符合 Decodable 协议
struct User: Codable {
let name: String
let age: Int
let isDeveloper: Bool

    // 自定义 CodingKeys
    enum CodingKeys: String, CodingKey {
        case name = "full_name"
        case age
        case isDeveloper = "is_developer"
    }

}

do {
let user = try JSONDecoder().decode(User.self, from: jsonData)
// 打印解析后的数据
print("Name: \(user.name)")
print("Age: \(user.age)")
print("Is Developer: \(user.isDeveloper)")
} catch {
print("解析失败: \(error.localizedDescription)")
}

let dictionary: [String: Any] = ["name": "John", "age": 30, "isDeveloper": true]
do {
// 将字典转换为 JSON 数据
let jsonData = try JSONSerialization.data(withJSONObject: dictionary, options: .prettyPrinted)
// 将 JSON 数据转换为字符串
if let jsonString = String(data: jsonData, encoding: .utf8) {
print(jsonString)
}
} catch {
print("字典转 JSON 失败: \(error.localizedDescription)")
}

let jsonString = """
{
"name": "John",
"age": 30,
"isDeveloper": true
}
"""

// 将 JSON 字符串转换为 Data
if let jsonData = jsonString.data(using: .utf8) {
print(jsonData)
}

// 将 model 转为 Dictionary
extension Encodable {
func toDictionary() -> [String: Any]? {
guard let data = try? JSONEncoder().encode(self) else {
return nil
}

        // 使用 JSONSerialization 将 Data 转换为字典
        return (try? JSONSerialization.jsonObject(with: data, options: [])) as? [String: Any]
    }

}

let user = User(name: "John", age: 30, isDeveloper: true)
if let userDictionary = user.toDictionary() {
print(userDictionary)
}
```

# 读validate-npm-package-name源码

这是我读的第一个npm的包validate-npm-package-name，相信大部分人看到这个包名大概就知道是干什么的了，没错，就是用来校验一个包的包名是否合法的。

validate-npm-package-name这个包导出一个`validate`方法，该方法接受一个包名作为参数传入，返回一个对象。具体用法如下：

```TypeScript
var validate = require("validate-npm-package-name")

/** ----包名合法---- */
validate("some-package")
validate("example.com")
// 返回
{
  validForNewPackages: true,
  validForOldPackages: true
}

/** ----包名不合法---- */
validate(" leading-space:and:weirdchars")
// 返回
{
  validForNewPackages: false,
  validForOldPackages: false,
  errors: [
    'name cannot contain leading or trailing spaces',
    'name can only contain URL-friendly characters'
  ]
}

```

那么如何实现上面功能呢？我们还是直接来看代码吧

```TypeScript
'use strict'

// 校验URL相关的字符
var scopedPackagePattern = new RegExp('^(?:@([^/]+?)[/])?([^/]+?)$')
// 主要是 nodejs内置模块的名称
var builtins = require('builtins')
// 保留名(黑名单)
var blacklist = [
  'node_modules',
  'favicon.ico'
]

// 导出一个 validate 方法，该方法接受一个参数 name
var validate = module.exports = function (name) {
  // 警告信息：过去允许、现在不允许的 兼容error
  var warnings = []
  // 存储不合法的包名的规则
  var errors = []

  /** -------------------------------- 一个合格报名应该遵守的规则 -------------------------------- */
  // 1. 包名不能为空
  if (name === null) {
    errors.push('name cannot be null')
    return done(warnings, errors)
  }

  if (name === undefined) {
    errors.push('name cannot be undefined')
    return done(warnings, errors)
  }

  // 2. 包名必须是字符串
  if (typeof name !== 'string') {
    errors.push('name must be a string')
    return done(warnings, errors)
  }

  // 3. 包名长度大于0
  if (!name.length) {
    errors.push('name length must be greater than zero')
  }

  // 4. 不能以 . _ 开头
  if (name.match(/^\./)) {
    errors.push('name cannot start with a period')
  }

  if (name.match(/^_/)) {
    errors.push('name cannot start with an underscore')
  }

  // 5. 两头不能有空格
  if (name.trim() !== name) {
    errors.push('name cannot contain leading or trailing spaces')
  }

  // No funny business
  // 6. 不能和nodejs中的 保留名称相同
  blacklist.forEach(function (blacklistedName) {
    if (name.toLowerCase() === blacklistedName) {
      errors.push(blacklistedName + ' is a blacklisted name')
    }
  })

  // Generate warnings for stuff that used to be allowed

  // core module names like http, events, util, etc
  // 校验包名是否是node 内置module名、给予警告
  builtins.forEach(function (builtin) {
    if (name.toLowerCase() === builtin) {
      warnings.push(builtin + ' is a core module name')
    }
  })

  // really-long-package-names-------------------------------such--length-----many---wow
  // the thisisareallyreallylongpackagenameitshouldpublishdowenowhavealimittothelengthofpackagenames-poch.
  // 8. 名称长度不能大于214
  if (name.length > 214) {
    warnings.push('name can no longer contain more than 214 characters')
  }

  // mIxeD CaSe nAMEs
  // 9. 必须全是小写，不能大小写都有
  if (name.toLowerCase() !== name) {
    warnings.push('name can no longer contain capital letters')
  }

  // 10. 不能有特殊字段 ~'!()*
  if (/[~'!()*]/.test(name.split('/').slice(-1)[0])) {
    warnings.push('name can no longer contain special characters ("~\'!()*")')
  }

  // 不能有 URL 特殊的符号
  // 包名不能包含non-url-safe字符
  if (encodeURIComponent(name) !== name) {
    // Maybe it's a scoped package name, like @user/package
    // 这里主要处理 scope package name 比如 @babel/core
    var nameMatch = name.match(scopedPackagePattern)
    if (nameMatch) {
      var user = nameMatch[1]
      var pkg = nameMatch[2]
      if (encodeURIComponent(user) === user && encodeURIComponent(pkg) === pkg) {
        return done(warnings, errors)
      }
    }

    errors.push('name can only contain URL-friendly characters')
  }

  return done(warnings, errors)
}

validate.scopedPackagePattern = scopedPackagePattern

/**
 * 
 * @param {*} warnings 对该包名的⚠️
 * @param {*} errors 对该包名的❌
 * @returns {
 *     validForNewPackages: true/false, 包名是否合法
 *     validForOldPackages: true/false, 兼容处理（以前的支持大小写混合，现在不支持）
 *     warnings: 警告信息
 *     errors: 错误信息，说明报名不合法
 * }
 */
var done = function (warnings, errors) {
  var result = {
    validForNewPackages: errors.length === 0 && warnings.length === 0,
    validForOldPackages: errors.length === 0,
    warnings: warnings,
    errors: errors
  }
  if (!result.warnings.length) delete result.warnings
  if (!result.errors.length) delete result.errors
  return result
}

```

看完了上面代码，会发现前面几个校验是直接返回了，而后面是先收集起来在同一返回，这是为什么呢？个人感觉是前三个判断说明这个包名格式就不对，肯定不合法，不需要进行后面的校验，而且后面的校验是针对字符串的，如果前面不返回，那么就会报错；后面收集起来是同一告诉用户有哪些规则不合法，方面统一改。



看完这个包，也学到了一个合法的包名应该有以下几点：

- 包名不能为空并且长度应该大于0

- 包名必须是字符串并且不能大小写混合，只能纯小写，而且名称长度不能大于214

- 包名不能以 . 或 _ 开头

- 包名两头不能有空格

- 包名不能与node 内置module名、保留名相同

- 不能有特殊字段 ~'!()* 和 包名不能包含non-url-safe字符



参考资料：

1. [https://zhuanlan.zhihu.com/p/362147023](https://zhuanlan.zhihu.com/p/362147023)

2. [https://github.com/npm/validate-npm-package-name](https://github.com/npm/validate-npm-package-name)


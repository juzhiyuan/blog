---
title: WordPress 插件：上传附件至阿里云 OSS
---

注意：插件已更新至 1.0.0 版本，本博文记录的是插件开发初期经历。

项目地址：https://github.com/juzhiyuan/wordpress-cdn

使用 WordPress 时希望将上传的附件存储在类似七牛云或阿里云 OSS 的服务中，目前网上找到的相关插件、在使用了一段时间后发现有如下几个问题：

1. 功能过多，我只希望加速访问附件，不需要加速全站静态资源；
2. 我希望将附件仅存储在七牛云或 OSS 中，但是某些插件会依旧存储在服务器中，只是通过使用 CDN 回源至附件上传目录的方式来实现加速；
3. 插件维护不积极；

于是自己便开发一款仅用于上传附件至相关服务的插件，过去经常使用七牛云做存储，但目前已有一款插件 wpqiniu 可以实现上传功能。

自己选择 OSS 作存储，如果该 OSS 和阿里云 ECS 在同一区域，可以使用内网域名加速上传。

开发该插件前需要考虑几件事：

1. 插件目录的结构组织；
2. 阿里云 OSS 上传、删除、查找功能相关的 SDK 调用方法；
3. 在插件激活时，应当向 wp_options 表中存储插件需要的相关字段；
4. 在激活插件后，应该暴露插件设置页入口到「设置」目录下；
5. 插件卸载时，应当将 wp_options 表中存储的插件相关字段移除；
6. 在通过「媒体库」上传附件至服务器后，应当读取上传的附件并再次上传至 OSS、删除本地附件；
7. 在上传附件时，如果是图片，应当将缩略图也上传至 OSS；
8. 当附件更新时，应该将更新后的附件上传至 OSS；
9. 在「媒体库」中删除附件时，应当删除远程 OSS 中的附件；
10. 在上传附件时，应检查远程 OSS 中是否有同目录、同文件名的文件，如果有，则应自动修改上传的文件名；

按照上面提到的点，对应的，我们可以方便地实现相关函数方法。

## 目录结构

参考[文档](https://codex.wordpress.org/zh-cn:%E5%BC%80%E5%8F%91%E4%B8%80%E4%B8%AA%E6%8F%92%E4%BB%B6)介绍，本插件 wordpress-oss 目录结构如下：

```md
.
 |-- actions.php  // 存放主要实现函数，被入口文件引用
 |-- LICENSE
 |-- README-cn.md
 |-- README.md
 |-- readme.txt  // 自述文件
 |-- sdk
 |   -- aliyun-oss/
 |-- uninstall.php  // 卸载时执行
 |-- views
 |   -- settings.php  // 设置页视图与逻辑
 -- wordpress-oss.php // 插件入口文件
```

## OSS SDK
阿里云 OSS 提供了 [PHP 版本的 SDK](https://help.aliyun.com/document_detail/32099.html) 供我们调用，使用方式也很简单：

#### 初始化

```php
require_once 'sdk/aliyun-oss/autoload.php';

use OSS\OssClient;
use OSS\Core\OssException;

$ossClient = new OssClient($accessKeyId, $accessKeySecret, $endpoint);
```

#### 上传
文件上传分为简单上传、文件上传等，该插件使用 uploadFile 方法实现文件上传。

```php
/** 
 * @param {String} bucket OSS存储空间
 * @param {String} object 文件名称
 * @param {String} filePath 文件路径
 */
$ossClient->uploadFile($bucket, $object, $filePath);
```

#### 删除

可使用 deleteObjects 批量删除文件

```php
/** 
 * @param {String} bucket OSS存储空间
 * @param {Array} deleteObjects 由文件名构成的字符串数组
 */
$ossClient->deleteObjects($bucket, $deleteObjects);
```

#### 判断文件是否需存在

```php
$ossClient->doesObjectExist($bucket, $object)
```

#### 插件激活/取消激活

WordPress 中提供了 register_activation_hook 钩子，用于在插件被激活时触发某些事件。我们希望此时它做一件事：

检查在 wp_options 表中，是否存在该插件配置相关的字段 wordpress_oss_options：若无，则创建该字段记录；若有，则更新 wp_options->upload_url_path 字段记录。

```php
// wordpress-oss.php
register_activation_hook(__FILE__, "wordpress_oss_activatition");
register_deactivation_hook(__FILE__, "wordpress_oss_deactivation");

// actions.php
function wordpress_oss_activatition()
{
  $options = get_option('wordpress_oss_options');

  if (!$options) {
    $options = array(
      'accessKeyId' => '',
      'accessKeySecret' => '',
      'endpoint' => '',
      'bucket' => '',
      'cdn_url_path' => '',
    );

    add_option('wordpress_oss_options', $options, '', 'yes');
  } else if (isset($options["cdn_url_path"]) && $options["cdn_url_path"] != "") {
    update_option('upload_url_path', $options['cdn_url_path']);
  }
}

function wordpress_oss_deactivation()
{
  $options = get_option('wordpress_oss_options');
  $options['cdn_url_path'] = get_option('upload_url_path');
  update_option('wordpress_oss_options', $options);
  update_option('upload_url_path', '');
}
```

#### 插件卸载

插件卸载后，应当清除 wp_options->wordpress_oss_options 记录、更新 wp_options->upload_url_path 值。

```php
<?php
if (!defined('WP_UNINSTALL_PLUGIN')) {
  // Exit when uninstall.php is not called from WordPress
  exit();
}

delete_option('wordpress_oss_options');
update_option('upload_url_path', '');
```

#### 上传附件至 OSS

过程是：文件类型过滤、获取 OSS 配置信息、实例化 OSS、上传文件、删除本地文件；

```php
// wordpress-oss.php
add_filter('wp_handle_upload', 'wordpress_oss_upload_attachment');

// actions.php
// 上传前文件过滤
function wordpress_oss_upload_attachment($upload)
{
  $mime_types = get_allowed_mime_types();
  $image_mime_types = array(
    $mime_types['jpg|jpeg|jpe'],
    $mime_types['gif'],
    $mime_types['png'],
    $mime_types['bmp'],
    $mime_types['tiff|tif'],
    $mime_types['ico'],
  );

  if (!in_array($upload['type'], $image_mime_types)) {
    $object = str_replace(wp_upload_dir()['basedir'] . '/', '', $upload['file']);
    $filePath = $upload['file'];
    wordpress_oss_file_upload($object, $filePath);
  }

  return $upload;
}

// 上传方法
function wordpress_oss_file_upload($object, $filePath)
{
  // Init OSS Client Instance
  $option = get_option('wordpress_oss_options');
  $ossClient = new OssClient($option['accessKeyId'], $option['accessKeySecret'], $option['endpoint']);

  // Action
  try {
    $ossClient->uploadFile($option['bucket'], $object, $filePath);

    wordpress_oss_delete_local_file($filePath);
  } catch (OssException $e) {
    return FALSE;
  }
}
```

#### 附件更新

```php
// wordpress-oss.php
// 附件（指缩略图）生成
add_filter('wp_generate_attachment_metadata', 'wordpress_oss_generate_attachment_metadata');

// 附件（指缩略图）更新
add_filter('wp_update_attachment_metadata', 'wordpress_oss_generate_attachment_metadata');

// actions.php
function wordpress_oss_generate_attachment_metadata($metadata)
{
  // 先上传主文件
  if (isset($metadata['file'])) {
    $attachment_key = $metadata['file'];
    $attachment_local_path = wp_upload_dir()['basedir'] . '/' . $attachment_key;
    wordpress_oss_file_upload($attachment_key, $attachment_local_path);
  }
  
  // 上传缩略图文件
  if (isset($metadata['sizes']) && count($metadata['sizes']) > 0) {
    foreach ($metadata['sizes'] as $val) {
      $attachment_thumb_key = dirname($metadata['file']) . '/' . $val['file'];
      $attachment_thumb_local_path = wp_upload_dir()['basedir'] . '/' . $attachment_thumb_key;

      wordpress_oss_file_upload($attachment_thumb_key, $attachment_thumb_local_path);
    }
  }

  return $metadata;
}
```

#### 删除附件

```php
// wordpress-oss.php
add_action('delete_attachment', 'wordpress_oss_delete_remote_attachment');

// actions.php
function wordpress_oss_delete_remote_attachment($post_id)
{
  // 需要被删除的文件名称数组
  $deleteObjects = array();
  $meta = wp_get_attachment_metadata($post_id);

  if (isset($meta['file'])) {
    $attachment_key = $meta['file'];
    array_push($deleteObjects, $attachment_key);
  } else {
    $file = get_attached_file($post_id);
    $attached_key = str_replace(wp_get_upload_dir()['basedir'] . '/', '', $file);
    $deleteObjects[] = $attached_key;
  }

  // 缩略图文件
  if (isset($meta['sizes']) && count($meta['sizes']) > 0) {
    foreach ($meta['sizes'] as $val) {
      $attachment_thumb_key = dirname($meta['file']) . '/' . $val['file'];
      array_push($deleteObjects, $attachment_thumb_key);
    }
  }

  if (!empty($deleteObjects)) {
    $option = get_option('wordpress_oss_options');
    $ossClient = new OssClient($option['accessKeyId'], $option['accessKeySecret'], $option['endpoint']);
    $ossClient->deleteObjects($option['bucket'], $deleteObjects);
  }
}
```

#### 添加设置页

```php
// wordpress-oss.php
add_action('admin_menu', 'add_settings_page');

// actions.php
function add_settings_page()
{
  require_once 'views/settings.php';
  add_options_page('WordPress OSS Settings', 'WordPress OSS', 'manage_options', 'wordpress-oss-plugin', 'generate_settings_page');
}
```

注意：form action 应当指向当前页地址，该地址在不同的 WordPress 版本中是不一样的。

```php
// views/settings.php

<?php
function generate_settings_page()
{
  if (!current_user_can('manage_options')) {
    wp_die(__("You don't have sufficient permissions to access the page."));
  }
  $options = get_option('wordpress_oss_options');
  if ($options && isset($_GET['_wpnonce']) && wp_verify_nonce($_GET['_wpnonce']) && !empty($_POST)) {
    if ($_POST['type'] == 'wordpress_oss_options_update') {
      $options = array(
        'accessKeyId' => (isset($_POST['accessKeyId'])) ? sanitize_text_field(trim(stripslashes($_POST['accessKeyId']))) : '',
        'accessKeySecret' => (isset($_POST['accessKeySecret'])) ? sanitize_text_field(trim(stripslashes($_POST['accessKeySecret']))) : '',
        'endpoint' => (isset($_POST['endpoint'])) ? sanitize_text_field(trim(stripslashes($_POST['endpoint']))) : '',
        'bucket' => (isset($_POST['bucket'])) ? sanitize_text_field(trim(stripslashes($_POST['bucket']))) : '',
        'cdn_url_path' => (isset($_POST['cdn_url_path'])) ? sanitize_text_field(trim(stripslashes($_POST['cdn_url_path']))) : '',
      );

      update_option('wordpress_oss_options', $options);

      update_option('upload_url_path', esc_url_raw(trim(trim(stripslashes($_POST['cdn_url_path'])))));
      ?>

      <div style="font-size: 25px;color: red; margin-top: 20px;font-weight: bold;">
        <p>WordPress OSS Saved!</p>
      </div>

    <?php
    }
  }
  ?>
  <div class="wrap">
    <h2>WordPress OSS Settings</h2>
    <p>Welcome to WordPress OSS plugin.</p>
    <form action="<?php echo wp_nonce_url('./options-general.php?page=' . WORDPRESS_OSS_BASE_FOLDER . '-plugin'); ?>" method="POST" id="wordpress-oss-form">
      <h3>
        <label for=""></label>accessKeyId:
        <input type="text" name="accessKeyId" value="<?php echo esc_attr($options['accessKeyId']) ?>" size="40" />
      </h3>

      <h3>
        <label for=""></label>accessKeySecret:
        <input type="text" name="accessKeySecret" value="<?php echo esc_attr($options['accessKeySecret']) ?>" size="40" />
      </h3>

      <h3>
        <label for=""></label>endpoint:
        <input type="text" name="endpoint" value="<?php echo esc_attr($options['endpoint']) ?>" size="40" />
      </h3>

      <h3>
        <label for=""></label>bucket:
        <input type="text" name="bucket" value="<?php echo esc_attr($options['bucket']) ?>" size="40" />
      </h3>

      <h3>
        <label for=""></label>cdn_url_path:
        <input type="text" name="cdn_url_path" value="<?php echo esc_attr($options['cdn_url_path']) ?>" size="40" />
      </h3>

      <p>
        <input type="submit" name="submit" value="Save" />
      </p>
      <input type="hidden" name="type" value="wordpress_oss_options_update">
    </form>
  </div>
<?php
}
?>
```

#### OSS 与 CDN 配置

1. OSS 保持为私有状态；
2. 授权 CDN 域名访问私有 OSS Bucket，并为 CDN 添加 Refer 防盗链。
3. 添加 RAM 子账户时，可使用 RAM 自定义策略实现：某个特定 RAM 账户管理某个特定 OSS，具体可惨叫 https://www.cloudcared.cn/2752.html。

```json
// 自定义策略
// oss-cloud-uat/pro 均为 OSS Bucket 名称
{
    "Version": "1",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "oss:*",
            "Resource": [
                "acs:oss:*:*:oss-cloud-uat",
                "acs:oss:*:*:oss-cloud-uat/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": "oss:ListObjects",
            "Resource": "acs:oss:*:*:oss-cloud-pro"
        },
        {
            "Effect": "Allow",
            "Action": "oss:GetObject",
            "Resource": "acs:oss:*:*:oss-cloud-pro/*"
        }
    ]
}
```

## 参考
- [wpqiniu](https://wordpress.org/plugins/wpqiniu/)

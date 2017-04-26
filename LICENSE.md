<?php

namespace U17\Common;

use U17\Core\Component\HttpKernel\Exception\HttpException;

/**
 * 
 * 模板操作类，参见U17原有的模板操作
 * 
 * @author GaoYanfei<efeil@163.com>
 * 
 */

class Template
{
	//模板数据
	private $params = array();

	//特定模板目录
	private $tpl_dir = array();

	//缓存目录
	private $cache_dir;

	private $tplfile;

	/**
	 * 初始化
	 *
	 * @param string $template_dir 模板目录配置
	 */
	public function __construct($template_dir)
	{
		$this->tpl_dir = $template_dir;

		$this->cache_dir = XX_ROOT . '/data';
	}

	public function assign($key, $value)
	{
		$this->params[$key] = $value;
	}

	/**
	 * 显示模板
	 *
	 * @param string $file 模板路径、相对于templates的相对路径
	 * @param string $dir 模板分组 当为admin时则为后台模板
	 * @param boolean $is_reduce 是否压缩
	 * @param string $category 模板类型
	 */
	public function display($dir, $file, $is_reduce=false, $category='tplcache')
	{
		$this->tplfile = $this->cache_dir.'/'.$category.'/'.$dir.'/'.$file . '.tpl.php';

		//如果模板缓存不存在则编译生成带目录
		if (!file_exists($this->tplfile)) {
			if (!$this->compile($file, $dir, $is_reduce,$category)) {
				throw new HttpException(500, "Not found the Template ($dir, $file)");
			}
		} else {
			if ($category=='tplcache' && isset($this->tpl_dir[$dir])) {
				$t1 = filemtime($this->tplfile);
				$t2 = filemtime($this->tpl_dir[$dir].'/'.$file.".html");

				if ($t2 > $t1) {
					if (!$this->compile($file, $dir, $is_reduce,$category)) {
						throw new HttpException(500, "Not found the Template ($dir, $file)");
					}
				}
			}
		}

		$this->_reader();
	}

	private function _reader()
	{
		extract($this->params);
		require $this->tplfile;
	}

	public function __toString(){
		return 'Template';
	}

	/**
	 * 生成模板缓存
	 *
	 * @param string $file 模板路径、相对于templates的相对路径
	 * @param string $dir 模板分组 当为admin时则为后台模板
	 * @param boolean $is_reduce 是否压缩
	 * @param string $category 模板类型
	 * @return 返回模版是否生成
	 */
	public function compile($file, $dir, $is_reduce=false, $category='tplcache')
	{
	    if ($category=='tplcache') {
			if (isset($this->tpl_dir[$dir])) {
				$template = file_get_contents($this->tpl_dir[$dir].'/'.$file.".html");
			} else {
				$sql = "SELECT t.temp_content AS content FROM #@__template t
					LEFT JOIN #@__template_dir d ON (d.dir_id=t.dir_id)
					WHERE t.is_use=1 AND t.temp_name=? AND d.dir_name=?";

				$db = DBUtil::newInstance()->master('u17');
	    		$template = $db->resultFirst($sql, array ($file, $dir));
			}
	    } else if($category=='special') {
			$sql = "select content from #@__special where is_use=1 and status=2 and name=? and path=?";

			//为了保证数据一致这里用主数据库
			$db = DBUtil::newInstance()->master('u17');
	    	$template = $db->resultValue($sql, array ($file, $dir));
		} else {
			throw new HttpException(500, "undefined category: {$category}");
		}

	    if ($template === false) {
	        return false;
	    }

	    $dir = dirname($this->tplfile);
		if (!file_exists($dir)) {
			mkdir($dir, 0777, true);
		}

	    file_put_contents($this->tplfile, $this->parse($template, $is_reduce));

	    return true;
	}

	/**
	 * 模板转换
	 *
	 * @param string 	$str 		要转换的模板内容
	 * @param int 		$istag 		是否是标记模板
	 * @param boolean 	$is_reduce 	是否压缩
	 * @return 转换后的模板
	 */
	private function parse($str, $is_reduce=false)
	{
	    //处理文件包含
	    $str = preg_replace ( "/\{template\s+([a-zA-Z0-9_\$\[\]]+)\s+([a-zA-Z0-9_\\/]+)\s*([a-zA-Z0-9_]*)\}/", "<?php \$this->display(\"\\1\",\"\\2\", (bool)\"\\3\"); ?>", $str );

	    //注释
	    $str = preg_replace( "/\{\/\/(.*)\}/" , "" , $str);

	    //处理if标记
	    $str = preg_replace ( "/\{if\s+(.+?)\}/", "<?php if(\\1) { ?>", $str );
	    $str = preg_replace ( "/\{else\}/", "<?php } else { ?>", $str );
	    $str = preg_replace ( "/\{elseif\s+(.+?)\}/", "<?php } elseif (\\1) { ?>", $str );
	    $str = preg_replace ( "/\{\/if\}/", "<?php } ?>", $str );

	    //处理foreach标记
	    $str = preg_replace ( "/\{foreach\s+([^\s^\}]+)\s+([^\s^\}]+)\}/", "<?php if(is_array(\\1)) foreach(\\1 AS \\2) { ?>", $str );
	    $str = preg_replace ( "/\{foreach\s+(\S+)\s+(\S+)\s+(\S+)\}/", "<?php if(is_array(\\1)) foreach(\\1 AS \\2 => \\3) { ?>", $str );
	    $str = preg_replace ( "/\{\/foreach\}/", "<?php } ?>", $str );

	    //处理函数
	    $str = preg_replace ( "/\{([a-zA-Z_\x7f-\xff][a-zA-Z0-9_\x7f-\xff:]*\(([^\{^\}]*)\))\}/", "<?php echo \\1;?>", $str );

	    //处理变量
	    $str = preg_replace ( "/\{(\\$[a-zA-Z_][a-zA-Z0-9_]*\-\>[a-zA-Z_][a-zA-Z0-9_]*\(([^\{^\}]*)\))\}/", "<?php echo \\1;?>", $str ); //类变量
	    $str = preg_replace ( "/\{(\\$[a-zA-Z_\x7f-\xff][a-zA-Z0-9_\x7f-\xff]*)\}/", "<?php echo \\1;?>", $str ); //变量

	    //数组变量
		$str = preg_replace_callback("/\{(\\$[a-zA-Z0-9_\[\]\'\"\$\x7f-\xff]+)\}/s", array($this, '_addquote'), $str);
		
        //对象变量
		$str = preg_replace_callback("/\{(\\$[a-zA-Z0-9_\-\>\$\x7f-\xff]+)\}/s", array($this, '_addquote'), $str);

	    //处理XML头
	    $str = preg_replace("/(\<\?xml.+\?\>)/","<?php echo '\\1';?>", $str);

	    //去掉多余空白
	    if ($is_reduce) {
	        $str = preg_replace( "/\s+/"," ",$str);
	    }

	    $str = "<?php defined('IN_XXCMS') or exit('Access Denied'); ?>" . $str;

	    return $str;
	}
	
	/**
	 * 变量数据加双相号
	 *
	 * @param string $var 变量数据
	 * @return 返回处理后的数据
	 */
	public function _addquote($match)
	{
		return "<?php echo " . str_replace("\\\"", "\"", preg_replace("/\[([a-zA-Z0-9_\-\.\x7f-\xff]+)\]/s", "['\\1']", $match[1])) . ";?>";
	}

}


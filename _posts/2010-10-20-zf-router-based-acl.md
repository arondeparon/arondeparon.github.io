---
published: true
layout: post
---

## Using Zend Framework’s router to setup ACL role-based controllers

Since I started using Zend Framework about 2.5 years ago, I have developed several applications in it, some of them using Zend_Acl.

Even though these applications work just fine, I have yet to discover an easy way to implement Zend_Acl in a uniform way and plugging it into my own framework so that I can easily set it up on future projects.

I am currently working on a application involving three user roles with very distinct features to them. I have first tried using [Zend_Acl](http://framework.zend.com/manual/en/zend.acl.html) to handle all of these permissions correctly, but have found it to be quite a pain. Most of my controllers end up containing three times the number of actions they would normally contain, or, even worse, I would have to implement several switches in one single action to facilitate the different features required by each of the user roles.

To make my life a bit easier, I have fiddled around with [Zend_Controller_Router_Route](http://framework.zend.com/manual/en/zend.controller.router.html) and have come up with a way to use one uniform URI scheme that points to different controllers, based on the user role. This is done without priorly setup routes, which saves me quite a lot of work.

All is done by extending Zend_Controller_Router_Route_Module and querying the ACL in the match() and assemble() methods.

    <?php 
     
    class App_Controller_Router_Route_Rolebasedcontroller extends Zend_Controller_Router_Route_Module
    {
        /**
         * Roles that should be rewritten automatically
         * 
         * @var array
         */
        protected $_rewriteRoles = array('employee', 'executive');
         
        /**
         * Exceptions that should not be rewritten
         * 
         * @var array
         */
        protected $_exceptions = array(
           array(
               'module' => 'auth',
               'controller' => 'index'
           ),
           array(
               'module' => 'core',
               'controller' => 'profile'
           )
        );
     
        /**
         * Matches a user submitted path. Assigns and returns an array of variables
         * on a successful match.
         *
         * If a request object is registered, it uses its setModuleName(),
         * setControllerName(), and setActionName() accessors to set those values.
         * Always returns the values as an array.
         *
         * @param string $path Path used to match against this routing map
         * @return array An array of assigned values or a false on a mismatch
         */
        public function match($path, $partial = false)
        {
            $result = parent::match($path, $partial);
             
            $role = Plano_Acl::getInstance()->getCurrentRole();
             
            if (null !== $role && in_array($role, $this->_rewriteRoles))
            {
                if (!$this->hasException($result['module'], $result['controller'], $result['action']))           
                {
                    if (isset($result[$this->_controllerKey]))
                    {
                        $result[$this->_controllerKey] = ucfirst($role) . '-' . ucfirst($result[$this->_controllerKey]);
                    }
                }
            }
             
            return $result;
        }
         
        /**
         * Assembles user submitted parameters forming a URL path defined by this route
         * Removes fole prefixes when required
         *
         * @param array $data An array of variable and value pairs used as parameters
         * @param bool $reset Weither to reset the current params
         * @return string Route path with user submitted parameters
         */
        public function assemble($data = array(), $reset = false, $encode = true, $partial = false)
        {
            if (!$this->_keysSet) {
                $this->_setRequestKeys();
            }
             
            $params = (!$reset) ? $this->_values : array();
     
            foreach ($data as $key => $value) {
                if ($value !== null) {
                    $params[$key] = $value;
                } elseif (isset($params[$key])) {
                    unset($params[$key]);
                }
            }
     
            $params += $this->_defaults;
     
            $url = '';
     
            if ($this->_moduleValid || array_key_exists($this->_moduleKey, $data)) {
                if ($params[$this->_moduleKey] != $this->_defaults[$this->_moduleKey]) {
                    $module = $params[$this->_moduleKey];
                }
            }
            unset($params[$this->_moduleKey]);
     
            $controller = $params[$this->_controllerKey];
             
            // remove role prefix from url when required
            $role = Plano_Acl::getInstance()->getCurrentRole();
             
            if (null !== $role && in_array($role, $this->_rewriteRoles))        
            {
                if (strtolower(substr($params[$this->_controllerKey], 0, strlen($role))) == strtolower($role))
                {
                    // Controller is in the form Role-Controller, so we should use strlen()+1 to extract the controller part
                    $controller = lcfirst(substr($params[$this->_controllerKey], strlen($role) + 1));
                }
            }
             
            unset($params[$this->_controllerKey]);
     
            $action = $params[$this->_actionKey];
            unset($params[$this->_actionKey]);
     
            foreach ($params as $key => $value) {
                $key = ($encode) ? urlencode($key) : $key;
                if (is_array($value)) {
                    foreach ($value as $arrayValue) {
                        $arrayValue = ($encode) ? urlencode($arrayValue) : $arrayValue;
                        $url .= '/' . $key;
                        $url .= '/' . $arrayValue;
                    }
                } else {
                    if ($encode) $value = urlencode($value);
                    $url .= '/' . $key;
                    $url .= '/' . $value;
                }
            }
     
            if (!empty($url) || $action !== $this->_defaults[$this->_actionKey]) {
                if ($encode) $action = urlencode($action);
                $url = '/' . $action . $url;
            }
     
            if (!empty($url) || $controller !== $this->_defaults[$this->_controllerKey]) {
                if ($encode) $controller = urlencode($controller);
                $url = '/' . $controller . $url;
            }
     
            if (isset($module)) {
                if ($encode) $module = urlencode($module);
                $url = '/' . $module . $url;
            }
     
            return ltrim($url, self::URI_DELIMITER);
        }   
         
        /**
         * Check wether the specified request is excempted
         * 
         * @param string $module
         * @param controller $controller
         * @param action $action
         * @return boolean
         */
        protected function hasException($module = null, $controller = null, $action = null)
        {
            $e = false;
             
            foreach ($this->_exceptions as $exception)
            {
                if (null !== $module)
                {
                    $e = (isset($exception['module']) && $exception['module'] == $module);
                    if (true === $e && null !== $controller)
                    {
                        $e = (isset($exception['controller']) && $exception['controller'] == $controller);
                         
                        if (true === $e && null !== $action)
                        {
                            if (isset($exception['action']))
                            {
                                $e = ($exception['action'] == $action);
                            }
                        }
                    }
                    if (true === $e) break;
                }
            }
             
            return $e;
        }
    }

---
published: true
layout: post
---

## Using Zend_CodeGenerator to generate a Data Mapper

I have always liked the Data Mapper pattern example [ZF Quickstart’s model creation](http://framework.zend.com/manual/en/learning.quickstart.create-model.html) section but have never actually used it in one of my applications.

The main reason is that I am lazy. Which, in this case, is a [good thing](http://www.codinghorror.com/blog/2005/08/how-to-be-lazy-dumb-and-successful.html). Applying the data mapper pattern to even a very simple database table would require sufficient amount of programming. Assuming every column should have at least a getter and a setter method, and it’s property defined in the model, this would take up 1 object property and 2 methods per column. Even with a relatively simple database table with, let’s say 6 columns, this would require me to write 6 properties and 12 methods. Asuming my application has 20 tables, this would mean I’d have to write 120 properties and 240 methods.

The worst thing is that these getters and setters don’t do anything special. It’s simply setting the property and returning the model, or getting the property. Doing this over and over is quite tedious and simply a waste of time.

So, being lazy and all, I have decided to make this job a bit easier by extending the Zend_Db_Table class with functionality that allows me to write the Mapper and Model for me. This was also a good moment to examine the capabilities of [Zend_CodeGenerator](http://framework.zend.com/manual/en/zend.codegenerator.html).

The code is at this time still a work in progress and not so generic that it can be copied straid into your application, but should it reach that point, I will make it available for download here. For now, I hope a short roundup of how I did can inspire you to do something similiar.

**Extending Zend_Db_Table:**

    <?php
     
    abstract class Plano_Db_Table_Abstract extends Zend_Db_Table
    {
        /**
         * Generate model by examining the DB structure (if not done already)
         *
         * @return void
         */
        public function setupMapperPattern()
        {
            $this->_createModel()
                ->_createMapper();
        }
     
        /**
         * Create model file
         *
         * @return Plano_Db_Table_Abstract
         */
        private function _createModel()
        {
            $reflectionClass = new ReflectionClass(get_class($this));
            if ($reflectionClass->isAbstract()) return false;
     
            $fields = $this->info(Zend_Db_Table::METADATA);
     
            $class = new Zend_CodeGenerator_Php_Class();
            $docblock = new Zend_CodeGenerator_Php_Docblock(array(
                'shortDescription' => 'Automatically generated data model',
                'longDescription' => 'This class has been automatically generated based on the dbTable "' . $this->info(Zend_Db_Table::NAME) . '" @ ' . strftime('%d-%m-%Y %H:%M')
            ));
     
            $modelClassName = preg_replace('/_DbTable/', '', get_class($this));
     
            $class->setName($modelClassName)
                ->setDocblock($docblock)
                ->setExtendedClass('Plano_Model_Abstract');
     
            $filter = new Zend_Filter_Word_UnderscoreToCamelCase();
     
            foreach ($fields as $key => $meta)
            {
                $propertyName = $filter->filter($key);
                $class
                    // Property
                    ->setProperty(array(
                        'name' => lcfirst($propertyName),
                        'visibility' => 'protected',
                        'defaultValue' => (isset($meta['DEFAULT']) && $meta['DEFAULT'] != '') ? $meta['DEFAULT'] : new Zend_CodeGenerator_Php_Property_DefaultValue("null"),
                        'docblock' => array(
                            'shortDescription' => 'Automatically generated from db'
                        )
                    ))
                    // Setter method
                    ->setMethod(array(
                        'name' => 'set' . ucfirst($propertyName),
                        'parameters' => array(
                            array('name' => 'value')
                        ),
                        'body' => '$this->' . lcfirst($propertyName) . ' = $value;' . "\n" . 'return $this;',
                        'docblock' => new Zend_CodeGenerator_Php_Docblock(array(
                            'shortDescription' => 'Set the ' . lcfirst($propertyName) . ' property',
                            'tags' => array(
                                new Zend_CodeGenerator_Php_Docblock_Tag_Param(array(
                                    'paramName' => 'value',
                                    'datatype' => 'string'
                                )),
                                new Zend_CodeGenerator_Php_Docblock_Tag_Return(array(
                                    'datatype' => $modelClassName
                                ))
                            )
                        ))
                    ))
                    // Getter method
                    ->setMethod(array(
                        'name' => 'get' . ucfirst($propertyName),
                        'body' => 'return $this->' . lcfirst($propertyName) . ';',
                        'docblock' => new Zend_CodeGenerator_Php_Docblock(array(
                            'shortDescription' => 'Get the ' . lcfirst($propertyName) . ' property',
                            'tags' => array(
                                new Zend_CodeGenerator_Php_Docblock_Tag_Return(array(
                                    'datatype' => 'string'
                                ))
                            )
                        ))
                    ));
            }
     
            $class->setProperty(array(
                'name' => 'installed',
                'visibility' => 'public',
                'docblock' => 'Installed flag. Remove to regenerate.',
                'defaultValue' => 1
            ));
     
            $front = Zend_Controller_Front::getInstance();
            $exploded = explode('_', $modelClassName);
            unset($exploded[0]); // module prefix
            unset($exploded[1]); // model prefix
            $model = implode('_', $exploded);
            $path = $front->getModuleDirectory() . DIRECTORY_SEPARATOR . 'models' . DIRECTORY_SEPARATOR . str_replace('_', DIRECTORY_SEPARATOR, $model) . '.php';
     
            $file = new Zend_CodeGenerator_Php_File(array(
                'classes' => array($class)
            ));
            file_put_contents($path, $file->generate());
     
            return $this;
        }
     
        /**
         * Create model file
         *
         * @return Plano_Db_Table_Abstract
         */
        private function _createMapper()
        {
            $reflectionClass = new ReflectionClass(get_class($this));
            if ($reflectionClass->isAbstract()) return false;
     
            $fields = $this->info(Zend_Db_Table::METADATA);
     
            $class = new Zend_CodeGenerator_Php_Class();
            $docblock = new Zend_CodeGenerator_Php_Docblock(array(
                'shortDescription' => 'Automatically generated data mapper',
                'longDescription' => 'This class has been automatically generated based on the dbTable "' . $this->info(Zend_Db_Table::NAME) . '" @ ' . strftime('%d-%m-%Y %H:%M')
            ));
     
            $modelClassName = preg_replace('/_DbTable/', '', get_class($this) . '_Mapper');
     
            $class->setName($modelClassName)
                ->setDocblock($docblock)
                ->setExtendedClass('Plano_Model_Mapper_Db');
     
            $filter = new Zend_Filter_Word_UnderscoreToCamelCase();
     
            $class->setProperty(array(
                'name' => '_dbTable',
                'defaultValue' => get_class($this),
                'visibility' => 'protected',
                'docblock' => array(
                    'shortDescription' => 'Db Table'
                )
            ));
     
            if (array_key_exists('deleted', $fields))
            {
                $class->setProperty(array(
                    'name' => '_deletedColumn',
                    'defaultValue' => 'deleted',
                    'visibility' => 'protected',
                    'docblock' => array(
                        'shortDescription' => 'Deleted column'
                    )
                ));
            }
     
            $front = Zend_Controller_Front::getInstance();
            $exploded = explode('_', $modelClassName);
            unset($exploded[0]); // module prefix
            unset($exploded[1]); // model prefix
            $model = implode('_', $exploded);
     
            $dir = $front->getModuleDirectory() . DIRECTORY_SEPARATOR . 'models' . DIRECTORY_SEPARATOR . $exploded[2];
     
            if (!is_dir($dir))
            {
                mkdir($dir, 0755, true);
            }
     
            $path = $dir . DIRECTORY_SEPARATOR . $exploded[3] . '.php';
     
            $file = new Zend_CodeGenerator_Php_File(array(
                'classes' => array($class)
            ));
            file_put_contents($path, $file->generate());
     
            return $this;
        }
      }
      
**What this code does:**

- First of all, it assumes you put the a DbTable class somewhere inside modules/yourmodule/models/DbTable/Class.php and you have the table name defined
- When extending from this base class you can call setupMapperPattern() which then creates the the model ‘Table.php’ and the mapper in ‘Table/Mapper.php’

**_Disclaimer_**: _this code is not intended to be copy pasted into your own application and will not work by default. The code could use some (thorough) cleaning up, is not as elegant as it could be and contains some hacky solutions, This post  is merely intended as an example of a practical use case for [ZendCodeGenerator](http://framework.zend.com/manual/en/zend.codegenerator.html).
_

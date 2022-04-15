1. 建表语句 基础字段

```python
__tablename__ = '表名'
id = Column(Integer, primary_key=True)
enable = Column(Boolean, default=True)
created_on = Column(DateTime, default=datetime.now,comment='创建时间')
changed_on = Column(DateTime, default=datetime.now, onupdate=datetime.now, nullable=True, comment='更新时间')
enable = Column(Boolean, default=True, comment='是否有效')

#格式化实体
 export_fields = ['id',  'created_on', 'enable', 'changed_on']

    def format_data(self, args=None):
        """格式化返回对象数据到接口"""
        info = {}
        if not args:
            args = self.export_fields
        for col in args:
            val = getattr(self, col, '')
            if col in ('created_on', 'changed_on'):
                val = val and val.strftime('%Y-%m-%d %H:%M:%S')
            info[col] = val
        return info
```



2.分页 list

```python

    if request.method.upper() == 'POST':
        params = request.json
    else:
        params = request.args
    keyword = params.get('keyword')
    page = int(params.get('page', 1))
    page_size = int(params.get('page_size', 20)) or 20
    m = User

    if keyword:
        p = keyword
        query = db.session.query(m).filter(or_(m.username.like('%{p}%'.format(p=p)),
                                               m.name.like('%{p}%'.format(p=p)),
                                               m.email.like('%{p}%'.format(p=p))
                                               )
                                           ).filter(m.enable==True)
        counts = get_objects_num(query)
        records = get_objects_by_page(query=query, model=m, page=page, page_size=page_size)
    else:
        counts = get_objects_num(model=m)
        records = get_objects_by_page(model=m, page=page, page_size=page_size)
    pages = counts // page_size + 1 if counts % page_size else counts // page_size
    records = [obj.format_data() for obj in records]
    return jsonify(msg='ok',
                   data={'records': records,
                         'pages': pages,
                         'counts': counts,
                         'page_size': page_size
                         },
                   code=200)
```

3.get 详情

```python
def teacher_get():
    if request.method == 'POST':
        params = request.json
    else:
        params = request.args

    teacherId = params.get('teacher_id')

    if not teacherId:
        return jsonify(msg='老师的ID参数错误', code=400)


    is_exist = db.session.query(Teacher).filter(Teacher.id == teacherId).first()
    if not is_exist:
        return jsonify(msg='无该老师详情', code=400)
    return jsonify(msg='查询老师成功！', data=is_exist.format_data(), code=200)
```



4.add  单例 增/改

```python
def teacher_add():
       # 参数
    if request.method == 'POST':
        params = request.json
    else:
        params = request.args
    log.info(params)

    userName = params.get('userName')
    age = params.get('age')
    teacher_id = params.get('teacher_id')

    try:
        if not teacher_id:
            entity = Teacher(username=userName, age=age)
            db.session.add(entity)
        else:
            db.session.query(Teacher).filter_by(id=teacher_id).update({
                #修改的参数
                'username': userName,
                'age': age
            })
            entity = db.session.query(Teacher).filter(Teacher.id == teacher_id).first()
    except Exception as e:
        log.exception(e)
        db.session.rolllback()
        return jsonify(msg="添加/修改老师失败，请联系管理员处理！", data={}, code=500)
    else:
        db.session.commit()
        return jsonify(msg='添加/修改老师成功！', data=entity.format_data(), code=200)
    finally:
        db.session.close()
```

4.删除

```python
def teacher_del():
    if request.method == 'POST':
        params = request.json
    else:
        params = request.args
    log.info(params)
    teacher_id = params.get('teacher_id')
    if not teacher_id:
        return jsonify(msg="请传id:123", ret=400)
    teacher = db.session.query(Teacher).filter(Teacher.id == teacher_id, Teacher.enable == True).first()
    if not teacher:
        return jsonify(msg="老师不存在:333", ret=400)
    try:
        db.session.query(Teacher).filter(Teacher.id == teacher_id).update({"enable": False})
        db.session.commit()
        return jsonify(msg="删除老师成功", ret=200)
    except Exception as e:
        log.exception(e)
        return jsonify(msg="删除失败,请联系管理员:53453", ret=500)
    finally:
        db.session.close()
```



5.批量新增

```python
def teacher_add_all():
    if request.method == 'POST':
        params = request.json
    else:
        params = request.args
    teacherList = params.get('data')
    resultList = []
    for col in teacherList:
        entity = Teacher(username=col.get('userName'), age=col.get('age'))
        resultList.append(entity)
    db.session.add_all(resultList)
    db.session.commit()
    db.session.close()
    return jsonify(msg="成功",ret=200)
```





```python
 exists()用法
 .scalar()用法
 if db.session.query(exists().where(PushGroup.name == name)).scalar():
            return jsonify(msg='该推送分组名已经存在，请确保分组名称唯一', code=401)
```


# doc.py
# -*- coding: utf-8 -*-
from extension import mysql
from flask import *
from flask.ext.login import login_required, current_user
import os

doc = Blueprint('doc', __name__, template_folder='views')

@doc.route('/doc/mqcp', methods=['GET','POST'])
@login_required
def doc_mqcp_route():
	productid=request.args.get('id')
	version=request.args.get('version')
	cur = mysql.connection.cursor()//连接数据库
	if version=="new":
		cur.execute('SELECT * FROM MQCP WHERE productid="%s" AND released=0;'%productid)
		if cur.fetchone() is not None:
			flash("In progress MQCP exists!", "error")
			return redirect("/product?id=%s&next="%productid)
		cur.execute('INSERT INTO MQCP (productid, filename) VALUES (%s, "Untitled MQCP");'%productid)
		mysql.connection.commit()
		cur.execute('SELECT version FROM MQCP WHERE productid=%s ORDER BY version DESC;'%productid)
		version=str(cur.fetchone()[0])
		#print version
		return redirect("/doc/mqcp?id=%s&version=%s"%(productid,version))
	docid=productid+' '+version
	if request.method=="POST":
		cur.execute('DELETE FROM MQCPInfo WHERE docid="%s";'%docid)
		submit=request.form.get('submit')
		if submit=="Copy":
			oldversion=request.form.get('version')
			olddocid=productid+' '+oldversion
			cur.execute('SELECT id FROM MQCPInfo ORDER BY id DESC;')
			maxid=cur.fetchone()[0]
			cur.execute('INSERT INTO MQCPInfo (no,multi,operations,referencedoc,no2,setuptime,optime,characteristics,ctq,location,dim,lowertol,uppertol,equipment,samplesize,docmethod,remark,productid) SELECT no,multi,operations,referencedoc,no2,setuptime,optime,characteristics,ctq,location,dim,lowertol,uppertol,equipment,samplesize,docmethod,remark,productid FROM MQCPInfo WHERE docid="%s";'%olddocid)
			cur.execute('UPDATE MQCPInfo SET docid="%s" WHERE id>%s;'%(docid,maxid))
			mysql.connection.commit()
		else:
			filename=request.form.get('filename')
			if filename=="":
				filename="Untitled MQCP"
			no=0
			for op in request.form.getlist('ops'):
				no+=10
				operations=request.form.get('operations[%s]'%op)
				refdoc=request.form.getlist('refdoc[%s]'%op)
				refdoc=",".join(refdoc)
				samplesize=request.form.get('samplesize[%s]'%op)
				docmethod=request.form.get('docmethod[%s]'%op)
				lines=request.form.getlist('lines[%s]'%op)
				multi=len(lines)
				i=0
				for line in lines:
					setuptime=request.form.get('setuptime[%s][%s]'%(op,line))
					optime=request.form.get('optime[%s][%s]'%(op,line))
					characteristics=request.form.get('characteristics[%s][%s]'%(op,line))
					ctq=request.form.get('ctq[%s][%s]'%(op,line))
					location=request.form.get('location[%s][%s]'%(op,line))
					dim=request.form.get('dim[%s][%s]'%(op,line))
					lowertol=request.form.get('lowertol[%s][%s]'%(op,line))
					uppertol=request.form.get('uppertol[%s][%s]'%(op,line))
					equipment=request.form.getlist('equipment[%s][%s]'%(op,line))
					equipment=",".join(equipment)
					remark=request.form.get('remark[%s][%s]'%(op,line))
					cur.execute('INSERT INTO MQCPInfo (docid,no,multi,operations,referencedoc,no2,setuptime,optime,characteristics,ctq,location,dim,lowertol,uppertol,equipment,samplesize,docmethod,remark,productid)  VALUES("%s","%s","%s","%s","%s","%d","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s");'%(docid,no,multi,operations,refdoc,i,setuptime,optime,characteristics,ctq,location,dim,lowertol,uppertol,equipment,samplesize,docmethod,remark,productid))
					i+=1
			cur.execute('UPDATE MQCP SET filename="%s" WHERE productid="%s" AND version="%s";'%(filename, productid, version))
			mysql.connection.commit()
			if submit!="Save":
				return render_template("MQCP.html", infos=[], len=0, lines=[], edit=True, select=[], copy=0, close=1)
	cur.execute('SELECT released, filename FROM MQCP WHERE productid="%s" AND version="%s";'%(productid,version))
	tmp=cur.fetchone()
	if not tmp:
		return render_template('404.html'), 404
	edit=tmp[0]
	filename=tmp[1]
	cur.execute('SELECT * FROM MQCPInfo WHERE docid="%s" ORDER BY id ASC;'%docid)
	tmp=cur.fetchall()
	op=-1
	infos=[]
	lines=[]
	for info in tmp:
		if info[6]==0:
			op+=1
			lines.append(info[3]-1)
		info+=(op,)
		infos.append(info)
		#print info
	if len(lines)==0:
		op=0
		lines=[0]
	cur.execute('SELECT version, filename FROM MQCP WHERE productid="%s" AND released=1;'%productid)
	select=cur.fetchall()
	//docu
	refdocFiles=os.listdir('static/uploads/%s/spec'%productid)
	refdocFiles2=os.listdir('static/uploads/%s/vgbmkb'%productid)
	refdocFiles3=os.listdir('static/uploads/%s/refdoc'%productid)
	cur.execute('SELECT id,name FROM Equipment WHERE productid="%s";'%productid)
	equipmentFiles=[equipment[0].replace('\n', ' ').replace('\r', '')+"-"+equipment[1].replace('\n', ' ').replace('\r', '') for equipment in cur.fetchall()]
	refdocFiles.insert(0,"- Select reference docs -")
	equipmentFiles.insert(0,"- Select equipments -")
	for file1 in refdocFiles:
		if file1.startswith('.'):
			print(file1)
			refdocFiles.remove(file1)
	for file1 in refdocFiles2:
		if file1.startswith('.'):
			print(file1)
			refdocFiles2.remove(file1)
	for file1 in refdocFiles3:
		if file1.startswith('.'):
			print(file1)
			refdocFiles3.remove(file1)
	return render_template("MQCP.html", productid=productid, filename=filename, infos=infos, len=op, lines=lines, edit=not edit, select=select, copy=len(select), 
							refdocFiles=refdocFiles, refdocFiles2=refdocFiles2, refdocFiles3=refdocFiles3, equipmentFiles=equipmentFiles, close=0)

@doc.route('/doc/processcard', methods=['GET','POST'])
@login_required
def doc_processcard_route():
	productid=request.args.get('id')
	version=request.args.get('version')
	cur = mysql.connection.cursor()
	if version=="new":
		cur.execute('SELECT * FROM ProcessCard WHERE productid="%s" AND released=0;'%productid)
		if cur.fetchone() is not None:
			flash("In progress Process Card exists!", "error")
			return redirect("/product?id=%s&next="%productid)
		cur.execute('INSERT INTO ProcessCard (productid, filename) VALUES (%s, "Untitled ProcessCard");'%productid)
		mysql.connection.commit()
		cur.execute('SELECT version FROM ProcessCard WHERE productid=%s ORDER BY version DESC;'%productid)
		version=str(cur.fetchone()[0])
		#print version
		return redirect("/doc/processcard?id=%s&version=%s"%(productid,version))
	docid=productid+' '+version
	if request.method=="POST":
		cur.execute('DELETE FROM ProcessCardInfo WHERE docid="%s";'%docid)
		submit=request.form.get('submit')
		if submit=="Copy":
			oldversion=request.form.get('version')
			olddocid=productid+' '+oldversion
			cur.execute('SELECT id FROM ProcessCardInfo ORDER BY id DESC;')
			maxid=cur.fetchone()[0]
			cur.execute('INSERT INTO ProcessCardInfo (dept,step,operation,spec,vgbmkb,equipment,program,material,wi,result,operator,comment,other,productid) SELECT dept,step,operation,spec,vgbmkb,equipment,program,material,wi,result,operator,comment,other,productid FROM ProcessCardInfo WHERE docid="%s";'%olddocid)
			cur.execute('UPDATE ProcessCardInfo SET docid="%s" WHERE id>%s;'%(docid,maxid))
			mysql.connection.commit()
		else:
			filename=request.form.get('filename')
			if filename=="":
				filename="Untitled ProcessCard"
			#print filename, productid, version
			step=0
			for op in request.form.getlist('ops'):
				dept=request.form.get('dept[%s]'%op)
				step+=10
				operation=request.form.get('operation[%s]'%op)
				spec=request.form.getlist('spec[%s]'%op)
				spec=",".join(spec)
				vgbmkb=request.form.getlist('vgbmkb[%s]'%op)
				vgbmkb=",".join(vgbmkb)
				equipment=request.form.getlist('equipment[%s]'%op)
				equipment=",".join(equipment)
				program=request.form.getlist('program[%s]'%op)
				program=",".join(program)
				material=request.form.getlist('material[%s]'%op)
				material=",".join(material)
				wi=request.form.getlist('wi[%s]'%op)
				wi=",".join(wi)
				result=request.form.get('result[%s]'%op)
				if result is None:
					result=0
				else:
					result=1
				operator=request.form.get('operator[%s]'%op)
				comment=request.form.get('comment[%s]'%op)
				other=request.form.get('other[%s]'%op)
				cur.execute('INSERT INTO ProcessCardInfo (docid,dept,step,operation,spec,vgbmkb,equipment,program,material,wi,result,operator,comment,other,productid) VALUES("%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s");'%(docid,dept,step,operation,spec,vgbmkb,equipment,program,material,wi,result,operator,comment,other,productid))
			cur.execute('UPDATE ProcessCard SET filename="%s" WHERE productid="%s" AND version="%s";'%(filename, productid, version))
			mysql.connection.commit()
			if submit!="Save":
				return render_template("processcard.html", infos=[], len=0, edit=True, select=[], copy=0, files=[], close=1)
	cur.execute('SELECT released, filename FROM ProcessCard WHERE productid="%s" AND version="%s";'%(productid,version))
	tmp=cur.fetchone()
	if not tmp:
		return render_template('404.html'), 404
	edit=tmp[0]
	filename=tmp[1]
	cur.execute('SELECT * FROM ProcessCardInfo WHERE docid="%s" ORDER BY id ASC;'%docid)
	tmp=cur.fetchall()
	op=0
	infos=[]
	for info in tmp:
		info+=(op,)
		infos.append(info)
		op+=1
	cur.execute('SELECT version, filename FROM ProcessCard WHERE productid="%s" AND released=1;'%productid)
	select=cur.fetchall()
	# cur.execute('SELECT name FROM Product WHERE productid="%s"'%productid)
	# name = cur.fetchone()
	specFiles=os.listdir('static/uploads/%s/spec'%productid)
	vgbmkbFiles=os.listdir('static/uploads/%s/vgbmkb'%productid)
	programFiles=os.listdir('static/uploads/%s/program'%productid)
	wiFiles=os.listdir('static/uploads/%s/wi'%productid)
	cur.execute('SELECT id,name FROM Equipment WHERE productid="%s";'%productid)
	equipmentFiles=[equipment[0].replace('\n', ' ').replace('\r', '')+"-"+equipment[1].replace('\n', ' ').replace('\r', '') for equipment in cur.fetchall()]
	cur.execute('SELECT id,name FROM Tooling WHERE productid="%s";'%productid)
	toolingFiles=[tooling[0].replace('\n', ' ').replace('\r', '')+"-"+tooling[1].replace('\n', ' ').replace('\r', '') for tooling in cur.fetchall()]
	specFiles.insert(0,"- Select spec files -")
	vgbmkbFiles.insert(0,"- Select vgbmkb files -")
	programFiles.insert(0,"- Select program files -")
	wiFiles.insert(0,"- Select wi files -")
	equipmentFiles.insert(0,"- Select equipments -")
	toolingFiles.insert(0,"- Select toolings -")
	for file1 in specFiles:
		if file1.startswith('.'):
			print(file1)
			specFiles.remove(file1)
	for file1 in vgbmkbFiles:
		if file1.startswith('.'):
			print(file1)
			vgbmkbFiles.remove(file1)
	for file1 in programFiles:
		if file1.startswith('.'):
			print(file1)
			programFiles.remove(file1)
	for file1 in wiFiles:
		if file1.startswith('.'):
			print(file1)
			wiFiles.remove(file1)
	return render_template("processcard.html", productid=productid, filename=filename, infos=infos, len=op, edit=not edit, select=select, copy=len(select), 
							specFiles=specFiles, vgbmkbFiles=vgbmkbFiles, programFiles=programFiles, wiFiles=wiFiles, equipmentFiles=equipmentFiles, toolingFiles=toolingFiles, close=0)

@doc.route('/doc/recordform', methods=['GET','POST'])
@login_required
def doc_recordform_route():
	productid=request.args.get('id')
	version=request.args.get('version')
	cur = mysql.connection.cursor()
	if version=="new":
		cur.execute('SELECT * FROM RecordForm WHERE productid="%s" AND released=0;'%productid)
		if cur.fetchone() is not None:
			flash("In progress Record Form exists!", "error")
			return redirect("/product?id=%s&next="%productid)
		cur.execute('INSERT INTO RecordForm (productid, filename) VALUES (%s, "Untitled RecordForm");'%productid)
		mysql.connection.commit()
		cur.execute('SELECT version FROM RecordForm WHERE productid=%s ORDER BY version DESC;'%productid)
		version=str(cur.fetchone()[0])
		#print version
		return redirect("/doc/recordform?id=%s&version=%s"%(productid,version))
	docid=productid+' '+version
	if request.method=="POST":
		cur.execute('DELETE FROM RecordFormInfo WHERE docid="%s";'%docid)
		submit=request.form.get('submit')
		if submit=="Copy":
			oldversion=request.form.get('version')
			olddocid=productid+' '+oldversion
			cur.execute('SELECT id FROM RecordFormInfo ORDER BY id DESC;')
			maxid=cur.fetchone()[0]
			cur.execute('INSERT INTO RecordFormInfo (serialno,partid,tableno,fixture,depthnominal,depthactual,widthmin,widthmax,go,nogo,cutterchange,offset,operator,opdate,remark,outerinner,productid) SELECT serialno,partid,tableno,fixture,depthnominal,depthactual,widthmin,widthmax,go,nogo,cutterchange,offset,operator,opdate,remark,outerinner,productid FROM RecordFormInfo WHERE docid="%s";'%olddocid)
			cur.execute('UPDATE RecordFormInfo SET docid="%s" WHERE id>%s;'%(docid,maxid))
			mysql.connection.commit()
		else:
			filename=request.form.get('filename')
			if filename=="":
				filename="Untitled RecordForm"
			no=0
			for op in request.form.getlist('ops'):
				no+=10
				serialno=(no+1)/2
				partid=request.form.get('partid[%s]'%op)
				tableno=request.form.get('tableno[%s]'%op)
				fixture=request.form.get('fixture[%s]'%op)
				depthnominal=request.form.get('depthnominal[%s]'%op)
				depthactual=request.form.get('depthactual[%s]'%op)
				widthmin=request.form.get('widthmin[%s]'%op)
				if widthmin is None:
					widthmin=0
				else:
					widthmin=1
				widthmax=request.form.get('widthmax[%s]'%op)
				if widthmax is None:
					widthmax=0
				else:
					widthmax=1
				go=request.form.get('go[%s]'%op)
				if go is None:
					go=0
				else:
					go=1
				nogo=request.form.get('nogo[%s]'%op)
				if nogo is None:
					nogo=0
				else:
					nogo=1
				cutterchange=request.form.get('cutterchange[%s]'%op)
				if cutterchange is None:
					cutterchange=0
				else:
					cutterchange=1
				offset=request.form.get('offset[%s]'%op)
				if offset is None:
					offset=0
				else:
					offset=1
				operator=request.form.get('operator[%s]'%op)
				opdate=request.form.get('opdate[%s]'%op)
				remark=request.form.get('remark[%s]'%op)
				outerinner=request.form.get('outerinner[%s]'%op)
				cur.execute('INSERT INTO RecordFormInfo (docid,serialno,partid,tableno,fixture,depthnominal,depthactual,widthmin,widthmax,go,nogo,cutterchange,offset,operator,opdate,remark,outerinner,productid)  VALUES("%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s");'%(docid,serialno,partid,tableno,fixture,depthnominal,depthactual,widthmin,widthmax,go,nogo,cutterchange,offset,operator,opdate,remark,outerinner,productid))
			cur.execute('UPDATE RecordForm SET filename="%s" WHERE productid="%s" AND version="%s";'%(filename, productid, version))
			mysql.connection.commit()
			if submit!="Save":
				return render_template("recordform.html", infos=[], len=0, edit=True, select=[], copy=0, close=1)
	cur.execute('SELECT released, filename FROM RecordForm WHERE productid="%s" AND version="%s";'%(productid,version))
	tmp=cur.fetchone()
	if not tmp:
		return render_template('404.html'), 404
	edit=tmp[0]
	filename=tmp[1]
	cur.execute('SELECT * FROM RecordFormInfo WHERE docid="%s" ORDER BY id ASC;'%docid)
	tmp=cur.fetchall()
	op=0
	infos=[]
	for info in tmp:
		info+=(op,)
		infos.append(info)
		#print info
		op+=1
	cur.execute('SELECT version, filename FROM RecordForm WHERE productid="%s" AND released=1;'%productid)
	select=cur.fetchall()
	return render_template("recordform.html", filename=filename, infos=infos, len=op, edit=not edit, select=select, copy=len(select), close=0)

@doc.route('/doc/workinstruction', methods=['GET','POST'])
@login_required
def doc_workinstruction_route():
	productid=request.args.get('id')
	version=request.args.get('version')
	cur = mysql.connection.cursor()
	if version=="new":
		cur.execute('SELECT * FROM WorkInstruction WHERE productid="%s" AND released=0;'%productid)
		if cur.fetchone() is not None:
			flash("In progress Work Instruction exists!", "error")
			return redirect("/product?id=%s&next="%productid)
		cur.execute('INSERT INTO WorkInstruction (productid, filename) VALUES (%s, "Untitled WorkInstruction");'%productid)
		mysql.connection.commit()
		cur.execute('SELECT version FROM WorkInstruction WHERE productid=%s ORDER BY version DESC;'%productid)
		version=str(cur.fetchone()[0])
		#print version
		return redirect("/doc/workinstruction?id=%s&version=%s"%(productid,version))
	docid=productid+' '+version
	if request.method=="POST":
		cur.execute('DELETE FROM WorkInstructionInfo WHERE docid="%s";'%docid)
		submit=request.form.get('submit')
		if submit=="Copy":
			oldversion=request.form.get('version')
			olddocid=productid+' '+oldversion
			cur.execute('SELECT id FROM WorkInstructionInfo ORDER BY id DESC;')
			maxid=cur.fetchone()[0]
			cur.execute('INSERT INTO WorkInstructionInfo (no,content,drawing,data,others,productid) SELECT no,content,drawing,data,others,productid FROM WorkInstructionInfo WHERE docid="%s";'%olddocid)
			cur.execute('UPDATE WorkInstructionInfo SET docid="%s" WHERE id>%s;'%(docid,maxid))
			mysql.connection.commit()
		else:
			filename=request.form.get('filename')
			if filename=="":
				filename="Untitled WorkInstruction"
			for op in request.form.getlist('ops'):
				no=request.form.get('no[%s]'%op)
				content=request.form.get('content[%s]'%op)
				drawing=request.form.get('drawing[%s]'%op)
				data=request.form.get('data[%s]'%op)
				others=request.form.get('others[%s]'%op)
				cur.execute('INSERT INTO WorkInstructionInfo (docid,no,content,drawing,data,others,productid)  VALUES("%s","%s","%s","%s","%s","%s","%s");'%(docid,no,content,drawing,data,others,productid))
			cur.execute('UPDATE WorkInstruction SET filename="%s" WHERE productid="%s" AND version="%s";'%(filename, productid, version))
			mysql.connection.commit()
			if submit!="Save":
				return render_template("workinstruction.html", infos=[], len=0, edit=True, select=[], copy=0, close=1)
	cur.execute('SELECT released, filename FROM WorkInstruction WHERE productid="%s" AND version="%s";'%(productid,version))
	tmp=cur.fetchone()
	if not tmp:
		return render_template('404.html'), 404
	edit=tmp[0]
	filename=tmp[1]
	cur.execute('SELECT * FROM WorkInstructionInfo WHERE docid="%s" ORDER BY id ASC;'%docid)
	tmp=cur.fetchall()
	op=0
	infos=[]
	for info in tmp:
		info+=(op,)
		infos.append(info)
		#print info
		op+=1
	cur.execute('SELECT version, filename FROM WorkInstruction WHERE productid="%s" AND released=1;'%productid)
	select=cur.fetchall()
	return render_template("workinstruction.html", filename=filename, infos=infos, len=op, edit=not edit, select=select, copy=len(select), close=0)


@doc.route('/doc/tooling', methods=['GET','POST'])
@login_required
def doc_tooling_route():
	productid=request.args.get('id')
	cur = mysql.connection.cursor()
	cur.execute('SELECT * FROM Product WHERE productid="%s";'%productid)
	product = cur.fetchone()
	if not product:
		return render_template('404.html'), 404
	if request.method=="POST":
		cur.execute('DELETE FROM Tooling WHERE productid="%s";'%productid)
		submit=request.form.get('submit')
		toolingid=0
		for op in request.form.getlist('ops'):
			toolingid+=1
			id=request.form.get('id[%s]'%op)
			name=request.form.get('name[%s]'%op)
			spec=request.form.get('spec[%s]'%op)
			cur.execute('INSERT INTO Tooling (productid,toolingid,id,name,specification)  VALUES("%s","%s","%s","%s","%s");'%(productid,toolingid,id,name,spec))
			mysql.connection.commit()
		if submit!="Save":
			return render_template("tooling.html", infos=[], len=0, close=1)
	cur.execute('SELECT * FROM Tooling WHERE productid="%s" ORDER BY toolingid ASC;'%productid)
	tmp=cur.fetchall()
	op=0
	infos=[]
	for info in tmp:
		info+=(op,)
		infos.append(info)
		#print info
		op+=1
	return render_template("tooling.html", infos=infos, len=op, close=0)

@doc.route('/doc/equipment', methods=['GET','POST'])
@login_required
def doc_equipment_route():
	productid=request.args.get('id')
	cur = mysql.connection.cursor()
	cur.execute('SELECT * FROM Product WHERE productid="%s";'%productid)
	product = cur.fetchone()
	if not product:
		return render_template('404.html'), 404
	if request.method=="POST":
		cur.execute('DELETE FROM Equipment WHERE productid="%s";'%productid)
		submit=request.form.get('submit')
		equipid=0
		for op in request.form.getlist('ops'):
			equipid+=1
			id=request.form.get('id[%s]'%op)
			name=request.form.get('name[%s]'%op)
			spec=request.form.get('spec[%s]'%op)
			cur.execute('INSERT INTO Equipment (productid,equipid,id,name,specification)  VALUES("%s","%s","%s","%s","%s");'%(productid,equipid,id,name,spec))
			mysql.connection.commit()
		if submit!="Save":
			return render_template("equipment.html", infos=[], len=0, close=1)
	cur.execute('SELECT * FROM Equipment WHERE productid="%s" ORDER BY equipid ASC;'%productid)
	tmp=cur.fetchall()
	op=0
	infos=[]
	for info in tmp:
		info+=(op,)
		infos.append(info)
		#print info
		op+=1
	return render_template("equipment.html", infos=infos, len=op, close=0)

@doc.route('/doc/dimensionalrecord', methods=['GET','POST'])
@login_required
def doc_mqcp_route():
	productid=request.args.get('id')
	version=request.args.get('version')
	cur = mysql.connection.cursor()
	if version=="new":
		cur.execute('SELECT * FROM DimensionalRecord WHERE productid="%s" AND released=0;'%productid)
		if cur.fetchone() is not None:
			flash("In progress dimensional inspection record exists!", "error")
			return redirect("/product?id=%s&next="%productid)
		cur.execute('INSERT INTO DimensionalRecord (productid, filename) VALUES (%s, "Untitled DimensionalRecord");'%productid)
		mysql.connection.commit()
		cur.execute('SELECT version FROM DimensionalRecord WHERE productid=%s ORDER BY version DESC;'%productid)
		version=str(cur.fetchone()[0])
		#print version
		return redirect("/doc/dimensionalrecord?id=%s&version=%s"%(productid,version))
	docid=productid+' '+version
	if request.method=="POST":
		cur.execute('DELETE FROM DimensionalRecordInfo WHERE docid="%s";'%docid)
		submit=request.form.get('submit')
		if submit=="Copy":
			oldversion=request.form.get('version')
			olddocid=productid+' '+oldversion
			cur.execute('SELECT id FROM DimensionalRecordInfo ORDER BY id DESC;')
			maxid=cur.fetchone()[0]
			cur.execute('INSERT INTO DimensionalRecordInfo (componame,unit,componum,drawingnum,rev,standard,qty,sketch,devalue1,devalue2,devalue3,devalue4,devalue5,devalue6,devalue7,devalue8,devalue9,devalue10,deviation1,deviation2,deviation3,deviation4,deviation5,deviation6,deviation7,deviation8,deviation9,deviation10,cutst,wequality,viquality,evaluation,inspector,qc,date1,date2,productid) SELECT componame,unit,componum,drawingnum,rev,standard,qty,sketch,devalue1,devalue2,devalue3,devalue4,devalue5,devalue6,devalue7,devalue8,devalue9,devalue10,deviation1,deviation2,deviation3,deviation4,deviation5,deviation6,deviation7,deviation8,deviation9,deviation10,cutst,wequality,viquality,evaluation,inspector,qc,date1,date2,productid FROM DimensionalRecordInfo WHERE docid="%s";'%olddocid)
			cur.execute('UPDATE DimensionalRecordInfo SET docid="%s" WHERE id>%s;'%(docid,maxid))
			mysql.connection.commit()
		else:
			filename=request.form.get('filename')
			if filename=="":
				filename="Untitled dimensional record"
			no=0
			for op in request.form.getlist('ops'):
				no+=10
				operations=request.form.get('operations[%s]'%op)
				refdoc=request.form.getlist('refdoc[%s]'%op)
				refdoc=",".join(refdoc)
				samplesize=request.form.get('samplesize[%s]'%op)
				docmethod=request.form.get('docmethod[%s]'%op)
				lines=request.form.getlist('lines[%s]'%op)
				multi=len(lines)
				i=0
				for line in lines:
					setuptime=request.form.get('setuptime[%s][%s]'%(op,line))
					optime=request.form.get('optime[%s][%s]'%(op,line))
					characteristics=request.form.get('characteristics[%s][%s]'%(op,line))
					ctq=request.form.get('ctq[%s][%s]'%(op,line))
					location=request.form.get('location[%s][%s]'%(op,line))
					dim=request.form.get('dim[%s][%s]'%(op,line))
					lowertol=request.form.get('lowertol[%s][%s]'%(op,line))
					uppertol=request.form.get('uppertol[%s][%s]'%(op,line))
					equipment=request.form.getlist('equipment[%s][%s]'%(op,line))
					equipment=",".join(equipment)
					remark=request.form.get('remark[%s][%s]'%(op,line))
					cur.execute('INSERT INTO MQCPInfo (docid,no,multi,operations,referencedoc,no2,setuptime,optime,characteristics,ctq,location,dim,lowertol,uppertol,equipment,samplesize,docmethod,remark,productid)  VALUES("%s","%s","%s","%s","%s","%d","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s","%s");'%(docid,no,multi,operations,refdoc,i,setuptime,optime,characteristics,ctq,location,dim,lowertol,uppertol,equipment,samplesize,docmethod,remark,productid))
					i+=1
			cur.execute('UPDATE MQCP SET filename="%s" WHERE productid="%s" AND version="%s";'%(filename, productid, version))
			mysql.connection.commit()
			if submit!="Save":
				return render_template("MQCP.html", infos=[], len=0, lines=[], edit=True, select=[], copy=0, close=1)
	cur.execute('SELECT released, filename FROM MQCP WHERE productid="%s" AND version="%s";'%(productid,version))
	tmp=cur.fetchone()
	if not tmp:
		return render_template('404.html'), 404
	edit=tmp[0]
	filename=tmp[1]
	cur.execute('SELECT * FROM MQCPInfo WHERE docid="%s" ORDER BY id ASC;'%docid)
	tmp=cur.fetchall()
	op=-1
	infos=[]
	lines=[]
	for info in tmp:
		if info[6]==0:
			op+=1
			lines.append(info[3]-1)
		info+=(op,)
		infos.append(info)
		#print info
	if len(lines)==0:
		op=0
		lines=[0]
	cur.execute('SELECT version, filename FROM MQCP WHERE productid="%s" AND released=1;'%productid)
	select=cur.fetchall()
	//docu
	refdocFiles=os.listdir('static/uploads/%s/spec'%productid)
	refdocFiles2=os.listdir('static/uploads/%s/vgbmkb'%productid)
	refdocFiles3=os.listdir('static/uploads/%s/refdoc'%productid)
	cur.execute('SELECT id,name FROM Equipment WHERE productid="%s";'%productid)
	equipmentFiles=[equipment[0].replace('\n', ' ').replace('\r', '')+"-"+equipment[1].replace('\n', ' ').replace('\r', '') for equipment in cur.fetchall()]
	refdocFiles.insert(0,"- Select reference docs -")
	equipmentFiles.insert(0,"- Select equipments -")
	for file1 in refdocFiles:
		if file1.startswith('.'):
			print(file1)
			refdocFiles.remove(file1)
	for file1 in refdocFiles2:
		if file1.startswith('.'):
			print(file1)
			refdocFiles2.remove(file1)
	for file1 in refdocFiles3:
		if file1.startswith('.'):
			print(file1)
			refdocFiles3.remove(file1)
	return render_template("MQCP.html", productid=productid, filename=filename, infos=infos, len=op, lines=lines, edit=not edit, select=select, copy=len(select), 
							refdocFiles=refdocFiles, refdocFiles2=refdocFiles2, refdocFiles3=refdocFiles3, equipmentFiles=equipmentFiles, close=0)

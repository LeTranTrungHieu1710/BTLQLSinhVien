create DATABASE QLSinhVien;
Go

use QLSinhVien;
go

create table Khoa(
	MaKhoa varchar(20) primary key,
	Tenkhoa nvarchar(100) not null,
	DienThoai varchar(15),
	Email varchar(100) unique
);

create table Nganh(
	Manganh varchar(20) primary key,
	Tennganh nvarchar(100) not null,
	Makhoa varchar(20),
	foreign key (Makhoa) references Khoa(Makhoa)
);

create table Lop(
	Malop varchar(20) primary key,
	Tenlop nvarchar(100) not null,
	Khoahoc varchar(20),
	MaNganh varchar(20),
	foreign key (MaNganh) references Nganh(MaNganh)
);

create table SinhVien(
	MaSV varchar(20) primary key,
	HoTen nvarchar(100) not null,
	NgaySinh Date,
	GioiTinh varchar(10),
	DiaChi nvarchar(255),
	SoDienThoai varchar(15),
	Email varchar(100) unique,
	NgayNhapHoc Date,
	TrangThai varchar(15),
	MaLop varchar(20),
	foreign key (MaLop) references Lop(MaLop)
);

create table GiangVien(
	MaGV varchar(20) primary key,
	HoTen varchar(15) not null,
	NgaySinh date,
	GioiTinh nvarchar(10),
	SoDienThoai varchar(15),
	Email varchar(100) unique,
	HocVi nvarchar(50),
	MaKhoa varchar(20),
	foreign key (MaKhoa) references Khoa(MaKhoa)
);

create table MonHoc(
	MaMH varchar (20) primary key,
	TenMH nvarchar(100) not null,
	SoTinChi int check (SoTinChi > 0),
	SoTiet int check (SoTiet >0),
	MaMonTienQuyet varchar(20),
	foreign key (MaMonTienQuyet) references MonHoc(MaMH)
);

create table HocKy(
	MaHk varchar(20) primary key,
	TenHocKy nvarchar(50),
	NamHoc varchar(20),
);

create table HocPhan(
	MaHP varchar(20) primary key,
	MaMH varchar(20),
	MaGv varchar(20),
	MaHK varchar(20),
	PhongHoc nvarchar(20),
	LichHoc nvarchar(20),
	SiSoToiDa int check (SiSoToiDa > 0),
	foreign key (MaMH) references MonHoc(MaMH),
	foreign key (MaGV) references GiangVien(MaGV),
	foreign key (MaHK) references HocKy(MaHK)
);

create table BangDangKy(
	MaDk varchar(20) primary key,
	MaSV varchar(20),
	MAHP varchar(20),
	NgayDangKy date default getdate(),
	TrangThai nvarchar(50),
	foreign key (MaSV) references SinhVien(MaSV),
	foreign key (MaHP) references HocPhan(MaHP)
);

create table Diem(
	MaDiem varchar(20) primary key,
	MaDK varchar(20),
	DiemQuaTrinh float check (DiemQuaTrinh >= 0 and DiemQuaTrinh <= 10),
	DiemThi float check (DiemThi >=0 and DiemThi <= 10),
	DiemTongKet float check (DiemTongKet >= 0 and DiemTongKet <= 10),
	XepLoai varchar(20),
	foreign key (MaDk) references BangDangKy(MaDk)
);
go

create procedure ThemSinhVien
	@MaSV varchar(20), @HoTen varchar(20),@NgaySinh Date,
	@GioiTinh varchar(10), @DiaChi varchar(20), @SoDienThoai varchar(15),
	@Email varchar(100), @NgayNhapHoc date, @TrangThai nvarchar(15), @MaLop varchar(20)
as
begin
	insert into SinhVien (MaSV, HoTen, NgaySinh, GioiTinh, DiaChi, SoDienThoai, Email, NgayNhapHoc, TrangThai, MaLop)
	values (@MaSV, @HoTen, @NgaySinh, @GioiTinh, @DiaChi, @SoDienThoai, @Email, @NgayNhapHoc, @TrangThai, @MaLop)
end;
go

create procedure CapNhat
	@MaSV varchar(20), @DiaChi nvarchar(255), @SoDienThoai varchar(15)
as
begin
	update SinhVien
	set DiaChi = @DiaChi, SoDienThoai = @SoDienThoai
	Where MaSV = @MaSV;
End;
go

create procedure XoaSinhVien
	@MaSV varchar(20)
as
begin
	delete from BangDangKy where MaSV = @MaSV;
	delete from SinhVien where MaSv = @MaSV;
end;
go

create view DanhSachSinhVienTheoLop as
select l.TenLop, sv.MaSV, sv.HoTen, sv.NgaySinh, sv.GioiTinh, sv.Email
from Lop l
join SinhVien sv on l.MaLop = sv.MaLop;
go

create view DanhSachMonHocCuaSinhVien as
select sv.MaSv, sv.HoTen, mh.MaMH, hp.MaHK
from SinhVien sv
join BangDangKy dk on sv.MaSV = dk.MaSv
join HocPhan hp on dk.MaHP = hp.MaHP
join MonHoc mh on hp.MaMH = mh.MaMH;
go

create view DiemTrungBinhSinhVien as
select sv.MaSV, sv.HoTen, round(avg(d.DiemTongKet), 2) as DiemTrungBinh
from SinhVien sv
join BangDangKy dk on sv.MaSV = dk.MaSV
join Diem d on dk.MaDK = d.MaDK
group by sv.MaSV, sv.HoTen;
go

create view ThongKeSinhVienTheoKhoa as
select k.MaKhoa, k.TenKhoa, count(sv.MaSV) as SoLuongSinhVien
from Khoa k
left join Nganh n on k.MaKhoa = n.MaKhoa
left join Lop l on n.MaNganh = l.MaNganh
left join SinhVien sv on l.MaLop = sv.MaLop
group by k.MaKhoa, k.TenKhoa;
go

create trigger trg_KiemTraSiSo
on BangDangKy
after insert
as
begin
	if exists (
		select 1
		from inserted i
		join HocPhan hp on i.MaHP = hp.MaHP
		where (select count(*) from BangDangKy dk where dk.MaHP = i.MaHP) > hp.SiSoToiDa
	)
	begin 
		raiserror (N'Sĩ số học phần đã đạt tối đa, không thể đăng ký thêm.', 16, 1);
		rollback transaction;
	end
end;
go

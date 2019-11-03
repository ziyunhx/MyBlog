title: 使用 EF Core CodeFirst 操作 Mysql 数据库
date: 2016-12-23
categories: 
- 技术
tags:
- mysql
- EFCore
- .NET

---

 EntityFramework(以下简称EF) 作为 .NET 广受欢迎的数据库操作中间件，支持了几乎所有你用过的关系型数据库，本文将非常基础的介绍其在 Mysql 中的使用。EF 常见的使用模式有三种：CodeFirst, ModelFirst, DBFirst；三种方式各有所特点，一般要根据实际的业务情况做选择。

<!--more-->
 在完全没有历史负担的情况下，选择 CodeFirst 更为普遍，在最新的 EntityFramework Core 中 CodeFirst 几乎是唯一的选择。本文将介绍如何使用 CodeFirst 创建和操作 Mysql 数据库，文中将以微信企业号的用户同步作为案例。IDE 使用 Visual Studio for Mac。

 首先，使用 Mac 版 Visual Studio 创建一个 Console Application(.NET Core) 项目：EFCoreSample。使用 Nuget 增加 EFCore 以及 Mysql 所需要的包：

 - Microsoft.EntityFrameworkCore
 - MySql.Data.EntityFrameworkCore


 然后，由于本次例子中会使用到微信企业号的 API，我们直接引用第三方包来实现：

 - Senparc.Weixin
 - Senparc.Weixin.QY


 第三步，在解决方案中新建一个 Models 文件夹，增加部门（Party），用户（User），标签（Tag）类，以及它们之间的相互关系部门标签（PartyTag），用户部门（UserParty），用户标签（UserTag）类，均为多对多关系。类的详细定义代码如下：


 ** 部门（Party） **

{% codeblock %}
public class Party
{
    public Party()
    {
        UserPartys = new List<UserParty>();
        PartyTags = new List<PartyTag>();
    }

    /// <summary>
    /// 部门Id
    /// </summary>
    [Key]
    public int PartyId { get; set; }
    /// <summary>
    /// 部门名称
    /// </summary>
    [Required]
    [StringLength(32)]
    public string Name { get; set; }
    /// <summary>
    /// 在父部门中的次序值。order值小的排序靠前。
    /// </summary>
    public int Order { get; set; }
    /// <summary>
    /// 上级部门Id
    /// </summary>
    public int ParentPartyId { get; set; }
    public List<UserParty> UserPartys { set; get; }
    public List<PartyTag> PartyTags { set; get; }
}
{% endcodeblock %}

 ** 用户（User） ** 

{% codeblock %}
public class User
{
    public User()
    {
        UserPartys = new List<UserParty>();
        UserTags = new List<UserTag>();
    }

    /// <summary>
    /// 员工UserID
    /// </summary>
    [Key]
    [StringLength(32)]
    public string UserId { get; set; }
    /// <summary>
    /// 头像url。注：小图将url最后的"/0"改成"/64"
    /// </summary>
    [StringLength(256)]
    public string Avatar { get; set; }
    /// <summary>
    /// 邮箱
    /// </summary>
    [StringLength(256)]
    public string Email { get; set; }
    /// <summary>
    /// 性别。gender=0表示男，=1表示女
    /// </summary>
    public int Gender { get; set; }
    /// <summary>
    /// 手机号码
    /// </summary>
    [StringLength(32)]
    public string Mobile { get; set; }
    /// <summary>
    /// 成员名称
    /// </summary>
    [StringLength(128)]
    public string Name { get; set; }
    /// <summary>
    /// 职位信息
    /// </summary>
    [StringLength(64)]
    public string Position { get; set; }
    /// <summary>
    /// 关注状态: 1=已关注，2=已冻结，4=未关注
    /// </summary>
    public int Status { get; set; }
    /// <summary>
    /// 微信号
    /// </summary>
    [StringLength(64)]
    public string Weixinid { get; set; }
    /// <summary>
    /// 成员所属部门列表
    /// </summary>
    public List<UserParty> UserPartys { get; set; }
    public List<UserTag> UserTags { set; get; }
}
{% endcodeblock %}

 ** 标签（Tag） ** 

{% codeblock %}
public class Tag
{
    public Tag()
    {
        PartyTags = new List<PartyTag>();
        UserTags = new List<UserTag>();
    }

    [Key]
    public int TagId { set; get; }
    [Required]
    [StringLength(64)]
    public string TagName { set; get; }
    public List<PartyTag> PartyTags { get; set; }
    public List<UserTag> UserTags { get; set; }
}
{% endcodeblock %}

 ** 部门标签（PartyTag） ** 

{% codeblock %}
public class PartyTag
{
    public int PartyId { set; get; }
    public Party Party { set; get; }

    public int TagId { set; get; }
    public Tag Tag { set; get; }
}
{% endcodeblock %}

 ** 用户部门（UserParty） ** 

{% codeblock %}
public class UserParty
{
    [StringLength(32)]
    public string UserId { set; get; }
    public User User { set; get; }

    public int PartyId { set; get; }
    public Party Party { set; get; }
}
{% endcodeblock %}

 ** 用户标签（UserTag） ** 

{% codeblock %}
public class UserTag
{
    [StringLength(32)]
    public string UserId { set; get; }
    public User User { set; get; }

    public int TagId { set; get; }
    public Tag Tag { set; get; }
}
{% endcodeblock %}

 第四步，增加 WechatContext 类来操作数据库：

{% codeblock %}
public class WechatContext : DbContext
{
    public DbSet<User> Users { set; get; }
    public DbSet<Party> Partys { set; get; }
    public DbSet<Tag> Tags { set; get; }
    public DbSet<PartyTag> PartyTags { set; get; }
    public DbSet<UserParty> UserPartys { set; get; }
    public DbSet<UserTag> UserTags { set; get; }

    protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
        => optionsBuilder.UseMySQL(@"server=127.0.0.1;user id=root;password=root;persistsecurityinfo=True;database=wechatdb;Character Set=utf8");

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<PartyTag>()
                    .HasKey(c => new { c.PartyId, c.TagId });
        modelBuilder.Entity<UserParty>()
                    .HasKey(c => new { c.PartyId, c.UserId });
        modelBuilder.Entity<UserTag>()
                    .HasKey(c => new { c.UserId, c.TagId });

    }
}
{% endcodeblock %}

 需要注意以下几点：1，修改服务器的IP与用户名和密码；2，不要忘记连接字符串中的“Character Set=utf8”，否则会出现中文乱码；3，OnModelCreating 中定义了几个关系表的联合主键，不可缺少。

 第五步，创建 UserHelper 类，放置到 Core 文件夹中，代码如下：

{% codeblock %}
/// <summary>
/// 用户帮助类，用于将用户组，Tag等信息转换到具体的用户
/// 该类可能在通知中用到
/// </summary>
public class UserHelper
{
    private static string corpID = "";
    private static string corpSecret = "";

    /// <summary>
    /// Get the wechat users and save to database.
    /// </summary>
    /// <returns></returns>
    public static bool GetWechatUserToDB()
    {
        string token = AccessTokenContainer.TryGetToken(corpID, corpSecret);
        WechatContext context = new WechatContext();

        //todo: 清空现有用户数据

        //获取微信企业号内的用户架构信息
        Dictionary<string, List<Tag>> _userTags = new Dictionary<string, List<Tag>>();
        Dictionary<int, List<Tag>> _partyTags = new Dictionary<int, List<Tag>>();

        Dictionary<string, User> Users = new Dictionary<string, User>();

        //查找所有Tag并插入数据库
        GetTagListResult tagList = MailListApi.GetTagList(token);

        if (tagList != null && tagList.taglist != null && tagList.taglist.Count > 0)
        {
            foreach (var tag in tagList.taglist)
            {
                int tagId = -1;
                if (Int32.TryParse(tag.tagid, out tagId))
                {
                    Tag tempTag = new Tag() { TagId = tagId, TagName = tag.tagname };

                    GetTagMemberResult tagMemberResult = MailListApi.GetTagMember(token, tagId);
                    if (tagMemberResult != null && tagMemberResult.partylist != null && tagMemberResult.partylist.Length > 0)
                    {
                        foreach (int party in tagMemberResult.partylist)
                        {
                            if (!_partyTags.ContainsKey(party))
                                _partyTags[party] = new List<Tag>();

                            _partyTags[party].Add(tempTag);
                        }
                    }

                    if (tagMemberResult != null && tagMemberResult.userlist != null && tagMemberResult.userlist.Count > 0)
                    {
                        foreach (var tagUser in tagMemberResult.userlist)
                        {
                            if (!_userTags.ContainsKey(tagUser.userid))
                                _userTags[tagUser.userid] = new List<Tag>();

                            _userTags[tagUser.userid].Add(tempTag);
                        }
                    }

                    context.Tags.Add(tempTag);
                }
            }
            context.SaveChanges();
        }

        //查找所有部门并插入数据库
        GetDepartmentListResult departmentList = MailListApi.GetDepartmentList(token);

        if (departmentList != null && departmentList.department != null)
        {
            foreach (var party in departmentList.department)
            {
                var tempParty = new Party() { PartyId = party.id, Name = party.name, Order = party.order, ParentPartyId = party.parentid };

                //此处需要查询所有的Tag保存到库中
                if (_partyTags.ContainsKey(party.id))
                {
                    tempParty.PartyTags = _partyTags[party.id].Select(f => new PartyTag() { PartyId = tempParty.PartyId, TagId = f.TagId }).ToList();
                }

                //根据部门查找所有用户并存入缓存
                GetDepartmentMemberInfoResult memberInfos = MailListApi.GetDepartmentMemberInfo(token, party.id, 1, 0);
                if (memberInfos != null && memberInfos.userlist != null && memberInfos.userlist.Count > 0)
                {
                    foreach (var member in memberInfos.userlist)
                    {
                        if (!Users.ContainsKey(member.userid))
                        {
                            Users[member.userid] = new User()
                            {
                                Avatar = member.avatar,
                                Email = member.email,
                                Gender = member.gender,
                                Mobile = member.mobile,
                                Name = member.name,
                                Position = member.position,
                                Status = member.status,
                                UserId = member.userid,
                                Weixinid = member.weixinid,
                                UserTags = (_userTags.ContainsKey(member.userid)&& _userTags[member.userid].Count > 0) ? _userTags[member.userid].Select(f => new UserTag() { UserId = member.userid, TagId = f.TagId }).ToList() : null                                   
                            };
                        }
                        Users[member.userid].UserPartys.Add(new UserParty() { PartyId = tempParty.PartyId, UserId = member.userid });
                    }
                }

                context.Partys.Add(tempParty);
            }
            context.SaveChanges();
        }

        if (Users != null && Users.Count > 0)
        {
            foreach(var user in  Users.Values)
                context.Users.Add(user);

            context.SaveChanges();
        }

        return true;
    }
}
{% endcodeblock %}
 
 代码中的 corpID 和 corpSecret 需要在微信企业号中获得，如果你没有企业号，那么将仅能创建一个空的数据库，而无法同步到任何数据。

 最后，在 Program 中调用数据库创建以及用户同步方法：

{% codeblock %}
class Program
{
    static void Main(string[] args)
    {
        //判断数据库是否存在,如果不存在则创建表
        WechatContext context = new WechatContext();

        if (!context.Database.EnsureCreated())
        {
            Console.WriteLine("Error: Unable to create the mysql database！");
        }
        else
        {
            Console.WriteLine("Create and Connect to MySQL success！");

            //初始化微信用户关系数据
            if (UserHelper.GetWechatUserToDB())
            {
                Console.WriteLine("Init wechat users to MySQL db success！");
            }
            else
            {
                Console.WriteLine("Error: Unable to Init wechat users to MySQL db！");
            }
        }
    }
}
{% endcodeblock %}

至此，一个简单的 Demo 就完成了，由于篇幅有限，许多地方都还有需要优化的地方，此处就不一一介绍了，完整代码可以在 [EFCoreSample](https://github.com/ziyunhx/Samples/tree/master/EFCoreSample) 查看。执行后你就可以在数据库看到结果了！
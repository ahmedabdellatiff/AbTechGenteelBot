# AbTechGenteelBot


using System;

using System.Collections.Generic;

using System.Configuration;

using System.Data;

using System.Data.SqlClient;

using System.IO;

using System.Linq;

using System.Net;

using System.Text;

using System.Threading.Tasks;

using System.Web;

namespace AbTechGenteelBot
{

    class Program
    {
        static void Main(string[] args)
        {
            //from App.Config get Notify Limit In Days
            int _NotifyLimit;
            if (!int.TryParse(ConfigurationManager.AppSettings["HbdNotifyLimitInDays"],
                out _NotifyLimit))
            {
                _NotifyLimit = 0;
            }
            //fetch data from database using db query 
            //recommend: using stored procedure  
            var FetchedMobiles = new List<string>();
            using (SqlConnection con =
                       new SqlConnection(ConfigurationManager.ConnectionStrings["DefaultConnection"].ConnectionString))
            {
                string TargetDay = DateTime.Now.AddDays(-1 * _NotifyLimit).ToString("MM-dd");
                SqlCommand cmd =
                    new SqlCommand(string.Format(@"SELECT Mobile FROM Employees
                                     WHERE CONVERT(VARCHAR(5),DateOfBirth,110)='{0}'
                                    ", TargetDay), con);
                con.Open();
                SqlDataReader reader = cmd.ExecuteReader();
                if (reader.HasRows)
                {
                    while (reader.Read())
                    {
                        FetchedMobiles.Add(reader[0].ToString());
                    }
                }
                reader.Close();
                con.Close();
            }
            //Send SMS to fetched mobile numbers
            //recommend: log fetched data in db table for example for debug and fix future problem
            //for example log mobile number and current datetime or creation datetime

            //Also recommend: get message from configurable source not fixed in code
            foreach (var item in FetchedMobiles)
            {
                LogSms(item);
                //SendSms(item, "HBD");
            }
            //in end we will publish this bot on Task Scheduler 

        }
        static void LogSms(string number)
        {
            using (SqlConnection con =
                      new SqlConnection(ConfigurationManager.ConnectionStrings["DefaultConnection"].ConnectionString))
            {
                SqlCommand cmd =
               new SqlCommand("INSERT INTO [dbo].[SmsLog](Mobile,CreationDateTime) Values(@Mobile,getdate())", con);
                cmd.Parameters.Add("@Mobile", SqlDbType.VarChar, 50).Value = number;
                con.Open();
                cmd.ExecuteNonQuery();
                con.Close();
            }
        }
        static void SendSms(string number, string message)
        {
            var username = ConfigurationManager.AppSettings["SmsUsername"];
            var password = ConfigurationManager.AppSettings["SmsPassword"];
            var custId = ConfigurationManager.AppSettings["SmsCustomerId"];
            var senderName = ConfigurationManager.AppSettings["SmsSenderText"];

            HttpWebRequest req = (HttpWebRequest)WebRequest.Create("https://www.smsbox.com/SMSGateway/Services/Messaging.asmx/Http_SendSMS");
            req.Method = "POST";
            req.ContentType = "application/x-www-form-urlencoded";
            string postData = string.Format("username={0}&password={1}&customerId={2}&senderText={3}&messageBody={4}&recipientNumbers={5}&defDate=&isBlink=false&isFlash=false",
               username, password, custId, senderName, HttpUtility.UrlEncode(message), number);
            req.ContentLength = postData.Length;

            StreamWriter stOut = new
            StreamWriter(req.GetRequestStream(),
            System.Text.Encoding.ASCII);
            stOut.Write(postData);
            stOut.Close();
            string strResponse;
            StreamReader stIn = new StreamReader(req.GetResponse().GetResponseStream());
            strResponse = stIn.ReadToEnd();
            stIn.Close();

        }
    }
}

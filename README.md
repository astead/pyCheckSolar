# pyCheckSolar
Roughly check Solar electric production vs expected output

I used to have a script that ran on my system daily and would send me alerts if something was wrong. I would like to re-do it and share it, but haven't decided exactly how I want it to work so I haven't really started yet.

The basic idea is that you check the weather for the amount of cloud cover for the day and the number of daylight hours and get a rough projection of solar production. If something is wrong it could send an email/text. It could also be integrated into an Alexa voice skill so you can ask how the solar is doing and it would let you know.

Below is my old code to do this as a Windows Service if anyone wants to use is.
When I get some time I'll finish the python version of this.

﻿using System;
using System.Collections.Generic;
using System.Net;
using System.Net.Mail;
using Newtonsoft.Json;
using System.Linq;
using System.Text;
using System.Threading.Tasks;

namespace CheckSolar
{
    class Program
    {
        static void Main(string[] args)
        {
            double peak_power = get_peak();
            double produced = get_produced();
            double expected = get_expected(peak_power);
            
            if (produced < expected)
            {
                // Something is wrong let's send an alert
                send_email_alert(produced, expected);
            }
        }

        static void send_email_alert(double produced, double expected)
        {
            string smtpAddress = ${{ secrets.SMTP_ADDRESS }};
            int portNumber = ${{ secrets.SMTP_PORT }};
            bool enableSSL = true;
            string emailFromAddress = ${{ secrets.EMAIL_ADDRESS }}; //Sender Email Address  
            string password = ${{ secrets.EMAIL_PASSWORD }; //Sender Password  
            string emailToAddress = ${{ secrets.EMAIL_ADDRESS }}; //Receiver Email Address  
            string subject = "Solar Production Alert";
            string body = "It looks like there may be something wrong with the solar panels." + Environment.NewLine +
                "Expected:"+expected+", Produced: "+produced+"." + Environment.NewLine +
                "URL to verify: https://monitoring.solaredge.com/solaredge-web/p/login";

            using (MailMessage mail = new MailMessage())
            {
                mail.From = new MailAddress(emailFromAddress);
                mail.To.Add(emailToAddress);
                mail.Subject = subject;
                mail.Body = body;
                mail.IsBodyHtml = true;
                using (SmtpClient smtp = new SmtpClient(smtpAddress, portNumber))
                {
                    smtp.Credentials = new NetworkCredential(emailFromAddress, password);
                    smtp.EnableSsl = enableSSL;
                    smtp.Send(mail);
                }
            }

        }

        static double get_expected(double peak_power)
        {
            double power_expected = -1;
            var url = "http://api.openweathermap.org/data/2.5/weather" +
                "?lat=" + ${{ secrets.LATITUDE }} + 
                "&lon=" + ${{ secrets.LONGITUDE }} + 
                "&APPID=" + ${{ secrets.OPENWEATHER_KEY }};
            WebClient w = new WebClient();
            String response = w.DownloadString(url);
            if (!String.IsNullOrEmpty(response))
            {
                Dictionary<string, object> my_obj = JsonConvert.DeserializeObject<Dictionary<string, object>>(response);
                Dictionary<string, long> my_clouds = JsonConvert.DeserializeObject<Dictionary<string, long>>(my_obj["clouds"].ToString());
                Dictionary<string, object> my_sys = JsonConvert.DeserializeObject<Dictionary<string, object>>(my_obj["sys"].ToString());
                long cloud_cover = (long)my_clouds["all"];

                // time is in seconds, divide by 60 to get it into minutes
                long day_light_time = ((long)my_sys["sunset"] - (long)my_sys["sunrise"])/60;

                // number of minutes of sunlight * peak power per minute (so divided by 60) * cloud cover factor
                power_expected = (double)((day_light_time) * (peak_power / 60.0) * ((100.0 - cloud_cover)/100.0));
                
                // Add 80 % tolerance
                power_expected = power_expected * 0.8;
            }
            return power_expected;
        }

        static double get_peak()
        {
            double value = -1;
            String day = DateTime.Today.ToString("yyyy-MM-dd");
            String url = "https://monitoringapi.solaredge.com/site/" +
                ${{ secrets.SOLAR_SITE_ID }} + "/details" +
                "?api_key=" + ${{ secrets.SOLAR_API_KEY }};

            WebClient w = new WebClient();
            String response = w.DownloadString(url);
            if (!String.IsNullOrEmpty(response))
            {
                Dictionary<string, object> my_obj = JsonConvert.DeserializeObject<Dictionary<string, object>>(response);
                Dictionary<string, object> my_obj2 = JsonConvert.DeserializeObject<Dictionary<string, object>>(my_obj["details"].ToString());
                value = (double)my_obj2["peakPower"];
                string status = my_obj2["status"].ToString();
            }
            return value;
        }

        static double get_produced()
        {
            double value = -1;
            String day = DateTime.Today.ToString("yyyy-MM-dd");
            String url = "https://monitoringapi.solaredge.com/site/" +
                ${{ secrets.SOLAR_SITE_ID }} + "/energy" +
                "?timeUnit=DAY" +
                "&endDate=" + day + 
                "&startDate=" + day + 
                "&api_key=" + ${{ secrets.SOLAR_API_KEY }};

            WebClient w = new WebClient();
            String response = w.DownloadString(url);
            if (!String.IsNullOrEmpty(response))
            {
                Dictionary<string, object> my_obj = JsonConvert.DeserializeObject<Dictionary<string, object>>(response);
                Dictionary<string, object> my_obj2 = JsonConvert.DeserializeObject<Dictionary<string, object>>(my_obj["energy"].ToString());
                List<Dictionary<string, object>> my_obj3 = JsonConvert.DeserializeObject<List<Dictionary<string, object>>>(my_obj2["values"].ToString());
                value = (double)my_obj3[0]["value"];
            }
            return value;
        }
    }
}

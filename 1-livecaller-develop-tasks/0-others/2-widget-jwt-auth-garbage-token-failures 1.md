How widget works now without logged in visitor:

without opening widget frame only widget api is sent:

1. https://api-widget.livecaller.io/v1/widget/?widget_id=c9929827-b253-41b6-a7b8-6aa741d58c37  - this request is sent which loads widget data: 
{
    "data": {
        "id": "c9929827-b253-41b6-a7b8-6aa741d58c37",
        "departments": [
            {
                "id": "5458a36d-f705-466b-8c57-be447450e5a0",
                "display_name": "Second Department",
                "time_group": {
                    "id": 3173,
                    "name": "24 \/ 7 Working hours",
                    "opening_hours": {
                        "friday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "monday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "sunday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "tuesday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "saturday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "thursday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "wednesday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "exceptions": []
                    },
                    "opening_hours_merged": {
                        "1": {
                            "name": "Mon - Sun",
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ],
                            "index": 1
                        }
                    },
                    "timezone": "Asia\/Tbilisi"
                },
                "is_open": true,
                "next_open": "2026-06-10T20:00:00.000000Z",
                "next_close": "2026-06-10T19:59:00.000000Z",
                "widget_id": null,
                "auto_assign": null,
                "extra_attributes": []
            },
            {
                "id": "b698a178-76fc-40b7-81ee-9a676c003137",
                "display_name": "Review Department",
                "time_group": {
                    "id": 3173,
                    "name": "24 \/ 7 Working hours",
                    "opening_hours": {
                        "friday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "monday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "sunday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "tuesday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "saturday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "thursday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "wednesday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "exceptions": []
                    },
                    "opening_hours_merged": {
                        "1": {
                            "name": "Mon - Sun",
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ],
                            "index": 1
                        }
                    },
                    "timezone": "Asia\/Tbilisi"
                },
                "is_open": true,
                "next_open": "2026-06-10T20:00:00.000000Z",
                "next_close": "2026-06-10T19:59:00.000000Z",
                "widget_id": null,
                "auto_assign": null,
                "extra_attributes": []
            },
            {
                "id": "b6bf4619-6bd5-4c42-9781-38baf7afaf07",
                "display_name": "default",
                "time_group": {
                    "id": 3173,
                    "name": "24 \/ 7 Working hours",
                    "opening_hours": {
                        "friday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "monday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "sunday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "tuesday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "saturday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "thursday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "wednesday": {
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ]
                        },
                        "exceptions": []
                    },
                    "opening_hours_merged": {
                        "1": {
                            "name": "Mon - Sun",
                            "hours": [
                                {
                                    "from": "00:00",
                                    "to": "23:59"
                                }
                            ],
                            "index": 1
                        }
                    },
                    "timezone": "Asia\/Tbilisi"
                },
                "is_open": true,
                "next_open": "2026-06-10T20:00:00.000000Z",
                "next_close": "2026-06-10T19:59:00.000000Z",
                "widget_id": null,
                "auto_assign": null,
                "extra_attributes": {
                    "translations": {
                        "ka": {
                            "display_name": "\u10e1\u10d0\u10d1\u10d0\u10df\u10dd"
                        }
                    }
                }
            }
        ],
        "features": [
            {
                "id": 1,
                "name": "call",
                "settings": {
                    "button_text": "Free Call"
                }
            },
            {
                "id": 2,
                "name": "chat",
                "settings": {
                    "button_text": "Online Chat"
                }
            },
            {
                "id": 3,
                "name": "branding",
                "settings": {
                    "img": null,
                    "url": "https:\/\/livecaller.io",
                    "title": "LiveCaller.io"
                }
            },
            {
                "id": 5,
                "name": "offline",
                "settings": {
                    "title": "",
                    "description": "Sorry, we are offline."
                }
            },
            {
                "id": 7,
                "name": "feedback",
                "settings": []
            }
        ],
        "fields": [
            {
                "order": "0",
                "required": true,
                "name": "name",
                "label": "Name",
                "type": "text",
                "values": null
            },
            {
                "order": 5,
                "required": true,
                "name": "pin_code",
                "label": "pin_code",
                "type": "text",
                "values": null
            }
        ],
        "callback_fields": [
            {
                "name": "phone",
                "label": "Phone Number",
                "input_type": "phone",
                "grid": {
                    "renderer": ""
                },
                "validator": {
                    "rules": "required|min:4|max:20",
                    "messages": [],
                    "custom_attributes": []
                }
            },
            {
                "name": "scheduled_at",
                "label": "Scheduled At",
                "input_type": "scheduled_at",
                "grid": {
                    "renderer": "date",
                    "options": {
                        "filter": "date"
                    }
                },
                "validator": {
                    "rules": "required|min:4|max:20",
                    "messages": [],
                    "custom_attributes": []
                }
            }
        ],
        "date_formats": {
            "default": {
                "format": "HH:mm",
                "date_format": "DD-MM-YYYY",
                "time_format": "HH:mm",
                "datetime_format": "DD-MM-YYYY HH:mm"
            },
            "conversations": {
                "show_diff": false
            }
        },
        "accepted_extensions": "avchd,avi,flv,mov,wmv,mp4,zip,rar,jpg,jpeg,png,doc,docx,xls,xlsx,pdf,gif,svg,doc,docx,txt,svg",
        "settings": {
            "theme": {
                "styles": ".lc-widget {--primary-color: #12ACD6;--secondary-color: #FFF;--text-color: #000;--base-background: linear-gradient(135deg, rgb(56, 98, 235) 0%, var(--primary-color) 100%);--box-background: linear-gradient(135deg, rgb(56, 98, 235) 0%, var(--primary-color) 100%);--base-color: var(--primary-color);--call-button: var(--secondary-color);--call-button-color: var(--primary-color);--call-button-chat: var(--primary-color);--call-button-in-white-background: var(--primary-color);--call-button-in-white-background-color: var(--secondary-color);--chat-button-in-background: var(--primary-color);--chat-button-in-colors: var(--primary-color);--chat-button: var(--primary-color);--send-message-bgColor: var(--primary-color);--title-color: var(--secondary-color);--description-color: var(--secondary-color);--hangup-button: #f76265;--control-button: var(--primary-color);--button-color: var(--secondary-color);--control-button-hover: var(--primary-color);--chat-button-in-index: var(--secondary-color);--receive-message-bgColor: #F4F6F8;--main-text-color: var(--secondary-color);--additional-text-color: var(--text-color);--back-background-color: rgba(0, 0, 0, 0.1);--error-background: #f76265;--close-button-background: rgba(0, 0, 0, 0.1);--close-button-background-department: rgba(0, 0, 0, 0.1);--close-button-icon-department: var(--secondary-color);--close-button-icon: var(--secondary-color);--close-button-background-light: rgba(0, 0, 0, 0.1);--theme-shadow-dark: rgba(76, 118, 255, 0.5);--theme-shadow-light: rgba(76, 118, 255, 0.3);}",
                "widget": [],
                "launcher": {
                    "data": {
                        "colors": {
                            "primary": "#12ACD6",
                            "background": "#fff"
                        }
                    },
                    "name": "Default",
                    "type": "component"
                },
                "variables": {
                    "colors": {
                        "text": "#000",
                        "primary": "#12ACD6",
                        "secondary": "#FFF"
                    },
                    "position": {
                        "side": "right"
                    }
                },
                "globalStyles": "@media (min-width: 768px) {.lc-frame-widget {right: 30px; bottom: 100px; max-height: calc(100vh - 150px);}}.lc-frame-launcher {position: fixed; right: 30px; bottom: 20px;}"
            },
            "initial_screen": "conversation",
            "message_typing_preview": true,
            "filter_departments_by_lang": false
        },
        "translations": {
            "ar": {
                "Bad": "\u0633\u064a\u0626",
                "Call": "\u0627\u062a\u0635\u0644",
                "Good": "\u062c\u064a\u062f",
                "Name": "\u0627\u0644\u0627\u0633\u0645",
                "Send": "\u0625\u0631\u0633\u0627\u0644",
                "Agree": "\u0645\u0648\u0627\u0641\u0642",
                "Allow": "\u0642\u0628\u0648\u0644",
                "Loved": "\u0631\u0627\u0626\u0639",
                "Cancel": "\u0625\u0644\u063a\u0627\u0621",
                "E-mail": "\u0627\u0644\u0628\u0631\u064a\u062f \u0627\u0644\u0625\u0644\u0643\u062a\u0631\u0648\u0646\u064a",
                "Friday": "\u0627\u0644\u062c\u0645\u0639\u0629",
                "Mobile": "\u0631\u0642\u0645 \u0627\u0644\u0647\u0627\u062a\u0641",
                "Monday": "\u0627\u0644\u0627\u062b\u0646\u064a\u0646",
                "Sunday": "\u0627\u0644\u0623\u062d\u062f",
                "Average": "\u0645\u062a\u0648\u0633\u0637",
                "Message": "\u0627\u0643\u062a\u0628 \u062a\u062c\u0631\u0628\u062a\u0643",
                "Tuesday": "\u0627\u0644\u062b\u0644\u0627\u062b\u0627\u0621",
                "Operator": "\u0627\u062d\u062f \u0645\u0645\u062b\u0644\u064a \u062e\u062f\u0645\u0629 \u0627\u0644\u062f\u0639\u0645",
                "Saturday": "\u0627\u0644\u0633\u0628\u062a",
                "Thursday": "\u0627\u0644\u062e\u0645\u064a\u0633",
                "Very Bad": "\u0633\u064a\u0626 \u062c\u062f\u0627\u064b",
                "Free Call": "\u0645\u0643\u0627\u0644\u0645\u0629 \u0635\u0648\u062a\u064a\u0629 \u0645\u062c\u0627\u0646\u064a\u0629",
                "Mute chat": "\u0643\u062a\u0645 \u0635\u0648\u062a \u0627\u0644\u0645\u062d\u0627\u062f\u062b\u0629",
                "Wednesday": "\u0627\u0644\u0627\u0631\u0628\u0639\u0627\u0621",
                "Start Chat": "\u0627\u0628\u062f\u0623 \u0627\u0644\u0645\u062d\u0627\u062f\u062b\u0629",
                "Choose Date": "\u0627\u062e\u062a\u0631 \u0627\u0644\u062a\u0627\u0631\u064a\u062e",
                "Choose Time": "\u0627\u062e\u062a\u0631 \u0627\u0644\u0648\u0642\u062a",
                "Online Chat": "\u0645\u062d\u0627\u062f\u062b\u0629 \u0643\u062a\u0627\u0628\u064a\u0629",
                "Unmute chat": "\u0625\u0644\u063a\u0627\u0621 \u0643\u062a\u0645 \u0635\u0648\u062a \u0627\u0644\u0645\u062d\u0627\u062f\u062b\u0629",
                "Working Hours": "\u0633\u0627\u0639\u0627\u062a \u0627\u0644\u0639\u0645\u0644",
                "Connecting ...": "\u062c\u0627\u0631\u064a \u0627\u0644\u0627\u062a\u0635\u0627\u0644 ...",
                "Add Your Number": "\u0627\u0636\u0641 \u0631\u0642\u0645\u0643",
                "Thanks for rate": "\u0634\u0643\u0631\u0627 \u0639\u0644\u0649 \u0627\u0644\u062a\u0642\u064a\u064a\u0645",
                "Allow Microphone": "\u0627\u0644\u0633\u0645\u0627\u062d \u0628\u0627\u0633\u062a\u062e\u062f\u0627\u0645 \u0627\u0644\u0645\u064a\u0643\u0631\u0648\u0641\u0648\u0646",
                "Request callback": "\u0637\u0644\u0628 \u0645\u0639\u0627\u0648\u062f\u0629 \u0627\u0644\u0627\u062a\u0635\u0627\u0644",
                "Back To Home Page": "\u0627\u0644\u0639\u0648\u062f\u0629 \u0625\u0644\u0649 \u0627\u0644\u0628\u062f\u0627\u064a\u0629",
                "Choose department": "\u0627\u062e\u062a\u0631 \u0627\u0644\u0642\u0633\u0645",
                "Type a message...": "\u0627\u0643\u062a\u0628 \u0631\u0633\u0627\u0644\u0629 ...",
                "Co-browsing request": "\u0637\u0644\u0628 \u0627\u0644\u062a\u0635\u0641\u062d \u0627\u0644\u0645\u0634\u062a\u0631\u0643",
                "Now, we are offline": "\u0646\u062d\u0646 \u063a\u064a\u0631 \u0645\u062a\u0627\u062d\u064a\u0646 \u0641\u064a \u0627\u0644\u0648\u0642\u062a \u0627\u0644\u062d\u0627\u0644\u064a",
                "Review conversation": "\u062a\u0642\u064a\u064a\u0645 \u0627\u0644\u0645\u062d\u0627\u062f\u062b\u0629",
                "You are in queue...": "\u0623\u0646\u062a \u0641\u064a \u0642\u0627\u0626\u0645\u0629 \u0627\u0644\u0627\u0646\u062a\u0638\u0627\u0631 ...",
                "Conversation is empty": "\u0627\u0644\u0645\u062d\u0627\u062f\u062b\u0629 \u0641\u0627\u0631\u063a\u0629",
                "One more step to call": "\u062e\u0637\u0648\u0629 \u0623\u062e\u0631\u0649 \u0644\u0644\u0627\u062a\u0635\u0627\u0644",
                "One more step to chat": "\u062e\u0637\u0648\u0629 \u0623\u062e\u0631\u0649 \u0644\u0625\u062c\u0631\u0627\u0621 \u0627\u0644\u0645\u062d\u0627\u062f\u062b\u0629",
                "Your message here ...": "\u0627\u0643\u062a\u0628 \u062a\u062c\u0631\u0628\u062a\u0643 \u0645\u0639 \u0641\u0631\u064a\u0642 \u0627\u0644\u062f\u0639\u0645 \u0647\u0646\u0627 ...",
                "How we are Helpfully ?": "\u0647\u0644 \u0627\u0646\u062a \u0633\u0639\u064a\u062f \u0628\u062e\u062f\u0645\u0629 \u0627\u0644\u062f\u0639\u0645\u061f",
                "Queue position: :position": "\u0645\u0643\u0627\u0646\u0643 \u0641\u064a \u0642\u0627\u0626\u0645\u0629 \u0627\u0644\u0627\u0646\u062a\u0638\u0627\u0631: :position",
                "Hi, we are here to help you": "\u0645\u0631\u062d\u0628\u0627\u064b\u060c \u0646\u062d\u0646 \u0647\u0646\u0627 \u0644\u0645\u0633\u0627\u0639\u062f\u062a\u0643",
                "Conversation will appear here": "\u0633\u062a\u0638\u0647\u0631 \u0627\u0644\u0645\u062d\u0627\u062f\u062b\u0629 \u0647\u0646\u0627",
                "Your conversation has ended !": "\u062a\u0645 \u0627\u0646\u0647\u0627\u0621 \u0645\u062d\u0627\u062f\u062b\u062a\u0643!",
                "please allow it to make a call.": "\u064a\u0631\u062c\u0649 \u0627\u0644\u0633\u0645\u0627\u062d \u0644\u0647\u0627 \u0628\u0625\u062c\u0631\u0627\u0621 \u0645\u0643\u0627\u0644\u0645\u0629",
                "Your conversation has finished !": "\u0627\u0646\u062a\u0647\u062a \u0645\u062d\u0627\u062f\u062b\u062a\u0643!",
                "Chosen time should be in the future": "\u0627\u0644\u0648\u0642\u062a \u0627\u0644\u0645\u062d\u062f\u062f \u064a\u062c\u0628 \u0623\u0646 \u064a\u0643\u0648\u0646 \u0641\u064a \u0627\u0644\u0645\u0633\u062a\u0642\u0628\u0644",
                "Maximum amount of callbacks reached": "\u062a\u0645 \u0627\u0644\u0648\u0635\u0648\u0644 \u0625\u0644\u0649 \u0627\u0644\u062d\u062f \u0627\u0644\u0623\u0642\u0635\u0649 \u0645\u0646 \u0637\u0644\u0628\u0627\u062a \u0627\u0639\u0627\u062f\u0629 \u0627\u0644\u0627\u062a\u0635\u0627\u0644",
                "The comment must be at least 3 characters.": "\u064a\u062c\u0628 \u0623\u0644\u0627 \u064a\u0642\u0644 \u0627\u0644\u062a\u0639\u0644\u064a\u0642 \u0639\u0646 3 \u0623\u062d\u0631\u0641",
                "You have not allowed microphone permission": "\u0644\u0645 \u062a\u0633\u0645\u062d \u0628\u0627\u0633\u062a\u062e\u062f\u0627\u0645 \u0627\u0644\u0645\u064a\u0643\u0631\u0648\u0641\u0648\u0646",
                "The Phone Number must be at least 4 characters.": "\u064a\u062c\u0628 \u0623\u0646 \u064a\u0643\u0648\u0646 \u0631\u0642\u0645 \u0627\u0644\u0647\u0627\u062a\u0641 4 \u0627\u0631\u0642\u0627\u0645 \u0639\u0644\u0649 \u0627\u0644\u0623\u0642\u0644",
                "Choose how you are comfortable with to contact us": "\u0627\u062e\u062a\u0631 \u0627\u0644\u0637\u0631\u064a\u0642\u0629 \u0627\u0644\u062a\u064a \u062a\u0646\u0627\u0633\u0628\u0643 \u0644\u0644\u0627\u062a\u0635\u0627\u0644 \u0628\u0646\u0627",
                "Your request about callback have successfully sent": "\u0627\u0633\u062a\u0642\u0628\u0644\u0646\u0627 \u0637\u0644\u0628\u0643 \u0628\u0634\u0623\u0646 \u0645\u0639\u0627\u0648\u062f\u0629 \u0627\u0644\u0627\u062a\u0635\u0627\u0644 \u0628\u0646\u062c\u0627\u062d",
                "To start secure co-browsing session please click allow": "\u0644\u0628\u062f\u0621 \u0627\u0644\u062a\u0635\u0641\u062d \u0627\u0644\u0645\u0634\u062a\u0631\u0643 \u0627\u0644\u0622\u0645\u0646 \u060c \u064a\u0631\u062c\u0649 \u0627\u0644\u0646\u0642\u0631 \u0641\u0648\u0642 \u0642\u0628\u0648\u0644",
                "Choose your preferred time and date for us to contact you": "\u0627\u062e\u062a\u0631 \u0627\u0644\u0648\u0642\u062a \u0648\u0627\u0644\u062a\u0627\u0631\u064a\u062e \u0627\u0644\u0645\u0641\u0636\u0644\u064a\u0646 \u0644\u0643 \u062d\u062a\u0649 \u0646\u062a\u0635\u0644 \u0628\u0643",
                "Fill the information to Start a conversation The team replies in a few hours": "\u0627\u0645\u0644\u0623 \u0627\u0644\u0645\u0639\u0644\u0648\u0645\u0627\u062a \u0644\u0628\u062f\u0621 \u0645\u062d\u0627\u062f\u062b\u0629\u060c \u064a\u0631\u062f \u0627\u0644\u0641\u0631\u064a\u0642 \u062e\u0644\u0627\u0644 \u0633\u0627\u0639\u0627\u062a \u0642\u0644\u064a\u0644\u0629",
                "The chat will be closed and the current conversation won't be accessible. You will be able to start a new conversation.": "\u0633\u064a\u062a\u0645 \u0625\u063a\u0644\u0627\u0642 \u0647\u0630\u0647 \u0627\u0644\u0645\u062d\u0627\u062f\u062b\u0629 \u0648\u0644\u0646 \u064a\u0645\u0643\u0646 \u0627\u0644\u0648\u0635\u0648\u0644 \u0625\u0644\u064a\u0647\u0627 \u0645\u062c\u062f\u062f\u0627\u064b. \u0633\u062a\u062a\u0645\u0643\u0646 \u0645\u0646 \u0628\u062f\u0621 \u0645\u062d\u0627\u062f\u062b\u0629 \u062c\u062f\u064a\u062f\u0629"
            },
            "en": {
                "Bad": "Bad",
                "Call": "Call",
                "Good": "Good",
                "Name": "Name",
                "Send": "Send",
                "Agree": "Agree",
                "Allow": "Allow",
                "Loved": "Loved",
                "Cancel": "Cancel",
                "E-mail": "E-mail",
                "Friday": "Friday",
                "Mobile": "Mobile",
                "Monday": "Monday",
                "Sunday": "Sunday",
                "Average": "Average",
                "Message": "Message",
                "Tuesday": "Tuesday",
                "Operator": "Operator",
                "Saturday": "Saturday",
                "Thursday": "Thursday",
                "Very Bad": "Very Bad",
                "Free Call": "Free Call",
                "Mute chat": "Mute chat",
                "Wednesday": "Wednesday",
                "Start Chat": "Start Chat",
                "Choose Date": "Choose Date",
                "Choose Time": "Choose Time",
                "Online Chat": "Online Chat",
                "Unmute chat": "Unmute chat",
                "Working Hours": "Working Hours",
                "Connecting ...": "Connecting ...",
                "Add Your Number": "Add your number",
                "Thanks for rate": "Thanks for your feedback",
                "Allow Microphone": "Allow microphone",
                "Request callback": "Request a callback",
                "Back To Home Page": "Back to home page",
                "Choose department": "Choose department",
                "Type a message...": "Type a message...",
                "Co-browsing request": "Co-browsing request",
                "Now, we are offline": "Now, we are offline",
                "Review conversation": "Review conversation",
                "You are in queue...": "You are in queue...",
                "Conversation is empty": "Conversation is empty",
                "One more step to call": "One more step to call",
                "One more step to chat": "One more step to chat",
                "Your message here ...": "Your message here ...",
                "How we are Helpfully ?": "How would you rate the support you received?",
                "Queue position: :position": "Queue position: :position",
                "Hi, we are here to help you": "Hi, we are here to help you",
                "Conversation will appear here": "Conversation will appear here",
                "Your conversation has ended !": "Your conversation has ended !",
                "please allow it to make a call.": "please allow it to make a call",
                "Your conversation has finished !": "Your conversation has finished !",
                "Chosen time should be in the future": "Chosen time should be in the future",
                "Maximum amount of callbacks reached": "Maximum amount of callbacks reached",
                "The comment must be at least 3 characters.": "The comment must be at least 3 characters.",
                "You have not allowed microphone permission": "You have not allowed microphone permission",
                "The Phone Number must be at least 4 characters.": "The phone number must be at least 4 characters.",
                "Choose how you are comfortable with to contact us": "Select how you would like to contact us",
                "Your request about callback have successfully sent": "Your request about callback has been submitted",
                "To start secure co-browsing session please click allow": "To start a secure co-browsing session please click allow",
                "Choose your preferred time and date for us to contact you": "Choose your preferred time and date for us to call you back",
                "Fill the information to Start a conversation The team replies in a few hours": "Fill in the information. The team will be in touch shortly",
                "The chat will be closed and the current conversation won't be accessible. You will be able to start a new conversation.": "The chat will be closed and the current conversation won't be accessible. You will be able to start a new conversation."
            },
            "es": {
                "Bad": "Malo",
                "Call": "Llamar",
                "Good": "Buena",
                "Name": "Nombre",
                "Send": "Enviar",
                "Agree": "Aceptar",
                "Allow": "Permitir",
                "Loved": "Me encanto",
                "Cancel": "Cancelar",
                "E-mail": "Email",
                "Friday": "Viernes",
                "Mobile": "M\u00f3vil",
                "Monday": "Lunes",
                "Sunday": "S\u00e1bado",
                "Average": "Promedio",
                "Message": "Mensaje",
                "Tuesday": "Martes",
                "Operator": "Operador",
                "Saturday": "S\u00e1bado",
                "Thursday": "Jueves",
                "Very Bad": "Muy mala",
                "Free Call": "Llama gratis",
                "Mute chat": "Silenciar chat",
                "Wednesday": "Mi\u00e9rcoles",
                "Start Chat": "Iniciar chat",
                "Choose Date": "Elegir fecha",
                "Choose Time": "Elegir hora",
                "Online Chat": "Chat en l\u00ednea",
                "Unmute chat": "No silenciar chat",
                "Working Hours": "Horas laborales",
                "Connecting ...": "Conectando",
                "Add Your Number": "Agrega tu n\u00famero",
                "Thanks for rate": "Gracias por calificar",
                "Allow Microphone": "Permitir micr\u00f3fono",
                "Request callback": "Solicitar devoluci\u00f3n de llamada",
                "Back To Home Page": "Regresar al inicio",
                "Choose department": "Seleccionar departamento",
                "Type a message...": "Escribiendo mensaje",
                "Co-browsing request": "Solicitud de co-navegaci\u00f3n",
                "Now, we are offline": "Estamos fuera de l\u00ednea",
                "Review conversation": "Calificar conversaci\u00f3n",
                "You are in queue...": "Estas en cola",
                "Conversation is empty": "La conversaci\u00f3n vacia",
                "One more step to call": "Un paso m\u00e1s para llamar",
                "One more step to chat": "Un paso m\u00e1s para el chat",
                "Your message here ...": "Tu mensaje aqu\u00ed",
                "How we are Helpfully ?": "\u00bfNuestra ayuda te fue \u00fatil?",
                "Queue position: :position": "Posici\u00f3n de la cola: :posici\u00f3n",
                "Hi, we are here to help you": "Hola estamos para ayudarte",
                "Conversation will appear here": "La conversaci\u00f3n aparecer\u00e1 aqu\u00ed.",
                "Your conversation has ended !": "\u00a1Tu conversaci\u00f3n ha terminado!",
                "please allow it to make a call.": "Por favor permita que haga una llamada",
                "Your conversation has finished !": "\u00a1Tu conversaci\u00f3n ha terminado!",
                "Chosen time should be in the future": "El tiempo elegido debe ser en el futuro.",
                "Maximum amount of callbacks reached": "Cantidad m\u00e1xima de devoluciones de llamada alcanzada",
                "The comment must be at least 3 characters.": "El comentario debe tener al menos 3 caracteres.",
                "You have not allowed microphone permission": "No ha permitido el permiso del micr\u00f3fono",
                "The Phone Number must be at least 4 characters.": "El n\u00famero de tel\u00e9fono debe tener al menos 4 caracteres.",
                "Choose how you are comfortable with to contact us": "Elija c\u00f3mo se siente c\u00f3modo para contactarnos",
                "Your request about callback have successfully sent": "Su solicitud de devoluci\u00f3n de llamada se ha enviado con \u00e9xito",
                "To start secure co-browsing session please click allow": "Para iniciar una sesi\u00f3n de navegaci\u00f3n conjunta segura, haga clic en Permitir",
                "Choose your preferred time and date for us to contact you": "Elige la hora y la fecha que prefieras para que nos pongamos en contacto contigo",
                "Fill the information to Start a conversation The team replies in a few hours": "Complete la informaci\u00f3n para Iniciar una conversaci\u00f3n El equipo responde en unas horas",
                "The chat will be closed and the current conversation won't be accessible. You will be able to start a new conversation.": "El chat se cerrar\u00e1 y no se podr\u00e1 acceder a la conversaci\u00f3n actual. Podr\u00e1s iniciar una nueva conversaci\u00f3n."
            },
            "hy": {
                "Bad": "\u054e\u0561\u057f",
                "Call": "\u0536\u0561\u0576\u0563\u0561\u0570\u0561\u0580\u0565\u0584",
                "Good": "\u053c\u0561\u057e",
                "Name": "\u0531\u0576\u0578\u0582\u0576",
                "Send": "\u0548\u0582\u0572\u0561\u0580\u056f\u0565\u056c",
                "Agree": "\u0540\u0561\u0574\u0561\u0571\u0561\u0575\u0576\u057e\u0565\u056c",
                "Allow": "\u0539\u0578\u0582\u0575\u056c\u0561\u057f\u0580\u0565\u056c",
                "Loved": "\u0547\u0561\u057f \u056c\u0561\u057e",
                "Cancel": "\u0549\u0565\u0572\u0561\u0580\u056f\u0565\u056c",
                "E-mail": "\u0537\u056c \u0563\u0580\u0561\u057c\u0578\u0582\u0574",
                "Friday": "\u0548\u0582\u0580\u0562\u0561\u0569",
                "Mobile": "\u0532\u057b\u057b\u0561\u0575\u056b\u0576\u056b \u0570\u0561\u0574\u0561\u0580",
                "Monday": "\u0535\u0580\u056f\u0578\u0582\u0577\u0561\u0562\u0569\u056b",
                "Sunday": "\u053f\u056b\u0580\u0561\u056f\u056b",
                "Average": "\u0544\u056b\u057b\u056b\u0576",
                "Message": "\u0540\u0561\u0572\u0578\u0580\u0564\u0561\u0563\u0580\u0578\u0582\u0569\u0575\u0578\u0582\u0576",
                "Tuesday": "\u0535\u0580\u0565\u0584\u0577\u0561\u0562\u0569\u056b",
                "Operator": "\u0555\u057a\u0565\u0580\u0561\u057f\u0578\u0580",
                "Saturday": "\u0577\u0561\u0562\u0561\u0569",
                "Thursday": "\u0570\u056b\u0576\u0563\u0577\u0561\u0562\u0569\u056b",
                "Very Bad": "\u0547\u0561\u057f \u057e\u0561\u057f",
                "Free Call": "\u0531\u0576\u057e\u0573\u0561\u0580 \u0566\u0561\u0576\u0563",
                "Mute chat": "\u0531\u0576\u057b\u0561\u057f\u0565\u056c \u0566\u0580\u0578\u0582\u0575\u0581\u0568",
                "Wednesday": "\u0579\u0578\u0580\u0565\u0584\u0577\u0561\u0562\u0569\u056b",
                "Start Chat": "\u054d\u056f\u057d\u0565\u0584 Chat-\u0568",
                "Choose Date": "\u0538\u0576\u057f\u0580\u0565\u0584 \u0531\u0574\u057d\u0561\u0569\u056b\u057e",
                "Choose Time": "\u0538\u0576\u057f\u0580\u0565\u0584 \u053a\u0561\u0574\u0561\u0576\u0561\u056f\u0568",
                "Online Chat": "\u0531\u057c\u0581\u0561\u0576\u0581 \u0536\u0580\u0578\u0582\u0581\u0561\u0580\u0561\u0576",
                "Unmute chat": "\u0544\u056b\u0561\u0581\u0576\u0565\u056c \u0566\u0580\u0578\u0582\u0575\u0581\u0568",
                "Working Hours": "\u0531\u0577\u056d\u0561\u057f\u0561\u0576\u0584\u0561\u0575\u056b\u0576 \u056a\u0561\u0574\u0565\u0580",
                "Connecting ...": "\u0544\u056b\u0561\u0581\u0578\u0582\u0574...",
                "Add Your Number": "\u0531\u057e\u0565\u056c\u0561\u0581\u0580\u0565\u055b\u0584 \u0541\u0565\u0580 \u0570\u0561\u0574\u0561\u0580\u0568",
                "Thanks for rate": "\u0547\u0576\u0578\u0580\u0570\u0561\u056f\u0561\u056c\u0578\u0582\u0569\u0575\u0578\u0582\u0576 \u0563\u0576\u0561\u0570\u0561\u057f\u0574\u0561\u0576 \u0570\u0561\u0574\u0561\u0580",
                "Allow Microphone": "\u0539\u0578\u0582\u0575\u056c\u0561\u057f\u0580\u0565\u056c \u056d\u0578\u057d\u0561\u0583\u0578\u0572\u0568",
                "Request callback": "\u054a\u0561\u0570\u0561\u0576\u057b\u0565\u056c \u0570\u0565\u057f\u0566\u0561\u0576\u0563",
                "Back To Home Page": "\u054e\u0565\u0580\u0561\u0564\u0561\u057c\u0576\u0561\u056c \u0533\u056c\u056d\u0561\u057e\u0578\u0580 \u0567\u057b",
                "Choose department": "\u0538\u0576\u057f\u0580\u0565\u0584 \u0562\u0561\u056a\u056b\u0576",
                "Type a message...": "\u0544\u0578\u0582\u057f\u0584\u0561\u0563\u0580\u0565\u0584 \u0570\u0561\u0572\u0578\u0580\u0564\u0561\u0563\u0580\u0578\u0582\u0569\u0575\u0578\u0582\u0576",
                "Co-browsing request": "\u0540\u0561\u0574\u0561\u057f\u0565\u0572 \u0566\u0576\u0576\u0561\u0580\u056f\u0578\u0582\u0574 \u056d\u0576\u0564\u0580\u0561\u0576\u0584",
                "Now, we are offline": "\u0531\u0575\u056a\u0574 \u0574\u0565\u0576\u0584 \u0561\u0576\u0581\u0561\u0576\u0581 \u0565\u0576\u0584",
                "Review conversation": "\u054e\u0565\u0580\u0561\u0576\u0561\u0575\u0565\u0584 \u056d\u0578\u057d\u0561\u056f\u0581\u0578\u0582\u0569\u0575\u0578\u0582\u0576\u0568",
                "You are in queue...": "\u0534\u0578\u0582\u0584 \u0570\u0565\u0580\u0569\u0578\u0582\u0574 \u0565\u0584...",
                "Conversation is empty": "\u0536\u0580\u0578\u0582\u0575\u0581\u0568 \u0564\u0561\u057f\u0561\u0580\u056f \u0567",
                "One more step to call": "\u0538\u0576\u057f\u0580\u0565\u0584 \u0570\u0561\u057b\u0578\u0580\u0564 \u0584\u0561\u0575\u056c\u0568",
                "One more step to chat": "\u0538\u0576\u057f\u0580\u0565\u0584 \u0570\u0561\u057b\u0578\u0580\u0564 \u0584\u0561\u0575\u056c\u0568",
                "Your message here ...": "\u0541\u0565\u0580 \u0570\u0561\u0572\u0578\u0580\u0564\u0561\u0563\u0580\u0578\u0582\u0569\u0575\u0578\u0582\u0576\u0576 \u0561\u0575\u057d\u057f\u0565\u0572...",
                "How we are Helpfully ?": "\u053b\u0576\u0579\u057a\u0565\u055e\u057d \u056f\u0563\u0576\u0561\u0570\u0561\u057f\u0565\u0584 \u0574\u0565\u0566",
                "Queue position: :position": "\u0540\u0565\u0580\u0569\u056b \u0564\u056b\u0580\u0584\u0568: :position",
                "Hi, we are here to help you": "\u0532\u0561\u0580\u0587, \u0574\u0565\u0576\u0584 \u0561\u0575\u057d\u057f\u0565\u0572 \u0565\u0576\u0584 \u0571\u0565\u0566 \u0585\u0563\u0576\u0565\u056c\u0578\u0582 \u0570\u0561\u0574\u0561\u0580",
                "Conversation will appear here": "\u053d\u0578\u057d\u0561\u056f\u0581\u0578\u0582\u0569\u0575\u0578\u0582\u0576\u0568 \u056f\u0570\u0561\u0575\u057f\u0576\u057e\u056b \u0561\u0575\u057d\u057f\u0565\u0572",
                "Your conversation has ended !": "\u0541\u0565\u0580 \u056d\u0578\u057d\u0561\u056f\u0581\u0578\u0582\u0569\u0575\u0578\u0582\u0576\u0576 \u0561\u057e\u0561\u0580\u057f\u057e\u0565\u056c \u0567",
                "please allow it to make a call.": "\u053d\u0576\u0564\u0580\u0578\u0582\u0574 \u0565\u0576\u0584 \u0569\u0578\u0582\u0575\u056c \u057f\u0561\u056c \u056d\u0578\u057d\u0561\u0583\u0578\u0572\u056b\u0576 \u0566\u0561\u0576\u0563 \u056f\u0561\u057f\u0561\u0580\u0565\u056c\u0578\u0582 \u0570\u0561\u0574\u0561\u0580",
                "Your conversation has finished !": "\u0541\u0565\u0580 \u056d\u0578\u057d\u0561\u056f\u0581\u0578\u0582\u0569\u0575\u0578\u0582\u0576\u0576 \u0561\u057e\u0561\u0580\u057f\u057e\u0561\u056e \u0567",
                "Chosen time should be in the future": "\u053a\u0561\u0574\u0561\u0576\u0561\u056f\u0568 \u057a\u0565\u057f\u0584 \u0567 \u056c\u056b\u0576\u056b \u0561\u057a\u0561\u0563\u0561\u0575\u0578\u0582\u0574",
                "Maximum amount of callbacks reached": "\u0540\u0561\u057d\u057e\u0565\u056c \u0567 \u0570\u0565\u057f\u0566\u0561\u0576\u0563\u0565\u0580\u056b \u0561\u057c\u0561\u057e\u0565\u056c\u0561\u0563\u0578\u0582\u0575\u0576 \u0584\u0561\u0576\u0561\u056f\u0568",
                "The comment must be at least 3 characters.": "\u0544\u0565\u056f\u0576\u0561\u0562\u0561\u0576\u0578\u0582\u0569\u0575\u0578\u0582\u0576\u0568 \u057a\u0565\u057f\u0584 \u0567 \u056c\u056b\u0576\u056b \u0561\u057c\u0576\u057e\u0561\u0566\u0576 3 \u0576\u056b\u0577",
                "You have not allowed microphone permission": "\u0534\u0578\u0582\u0584 \u0569\u0578\u0582\u0575\u056c \u0579\u0565\u0584 \u057f\u057e\u0565\u056c \u056d\u0578\u057d\u0561\u0583\u0578\u0572\u056b \u0569\u0578\u0582\u0575\u056c\u057f\u057e\u0578\u0582\u0569\u0575\u0578\u0582\u0576",
                "The Phone Number must be at least 4 characters.": "\u0540\u0565\u057c\u0561\u056d\u0578\u057d\u0561\u0570\u0561\u0574\u0561\u0580\u0568 \u057a\u0565\u057f\u0584 \u0567 \u056c\u056b\u0576\u056b \u0561\u057c\u0576\u057e\u0561\u0566\u0576 4 \u0576\u056b\u0577",
                "Choose how you are comfortable with to contact us": "\u0538\u0576\u057f\u0580\u0565\u0584, \u056b\u0576\u0579\u057a\u0565\u057d \u0565 \u0570\u0561\u0580\u0574\u0561\u0580 \u0574\u0565\u0566 \u0570\u0565\u057f \u056f\u0561\u057a\u057e\u0565\u056c",
                "Your request about callback have successfully sent": "\u0540\u0565\u057f\u0561\u0564\u0561\u0580\u0571 \u0566\u0561\u0576\u0563\u056b \u057e\u0565\u0580\u0561\u0562\u0565\u0580\u0575\u0561\u056c \u0571\u0565\u0580 \u0570\u0561\u0580\u0581\u0578\u0582\u0574\u0568 \u0570\u0561\u057b\u0578\u0572\u0578\u0582\u0569\u0575\u0561\u0574\u0562 \u0578\u0582\u0572\u0561\u0580\u056f\u057e\u0565\u056c \u0567",
                "To start secure co-browsing session please click allow": "\u0531\u0576\u057e\u057f\u0561\u0576\u0563 \u0570\u0561\u0574\u0561\u057f\u0565\u0572 \u0566\u0576\u0576\u0561\u0580\u056f\u0578\u0582\u0574 \u057d\u056f\u057d\u0565\u056c\u0578\u0582 \u0570\u0561\u0574\u0561\u0580 \u057d\u0565\u0572\u0574\u0565\u0584 \u00ab\u0539\u0578\u0582\u0575\u056c\u0561\u057f\u0580\u0565\u056c\u00bb:",
                "Choose your preferred time and date for us to contact you": "\u0538\u0576\u057f\u0580\u0565\u0584,\u0570\u0561\u0580\u0574\u0561\u0580 \u056a\u0561\u0574\u0568 \u0587 \u0561\u0574\u057d\u0561\u0569\u056b\u057e\u0568 \u0578\u0580\u057a\u0565\u057d\u0566\u056b \u0574\u0565\u0576\u0584 \u0571\u0565\u0566 \u0570\u0565\u057f \u056f\u0561\u057a\u057e\u0565\u0576\u0584",
                "Fill the information to Start a conversation The team replies in a few hours": "\u053c\u0580\u0561\u0581\u0580\u0565\u0584 \u057f\u0565\u0572\u0565\u056f\u0561\u057f\u057e\u0578\u0582\u0569\u0575\u0578\u0582\u0576\u0568 \u0587 \u0574\u0565\u0580 \u0569\u056b\u0574\u0568 \u0577\u0578\u0582\u057f\u0578\u057e \u056f\u057a\u0561\u057f\u0561\u057d\u056d\u0561\u0576\u056b",
                "The chat will be closed and the current conversation won't be accessible. You will be able to start a new conversation.": "\u0536\u0580\u0578\u0582\u0575\u0581\u0568 \u056f\u0583\u0561\u056f\u057e\u056b, \u0587 \u0568\u0576\u0569\u0561\u0581\u056b\u056f \u056d\u0578\u057d\u0561\u056f\u0581\u0578\u0582\u0569\u0575\u0578\u0582\u0576\u0568 \u0570\u0561\u057d\u0561\u0576\u0565\u056c\u056b \u0579\u056b \u056c\u056b\u0576\u056b.\u0534\u0578\u0582\u0584 \u056f\u056f\u0561\u0580\u0578\u0572\u0561\u0576\u0561\u0584 \u0576\u0578\u0580 \u056d\u0578\u057d\u0561\u056f\u0581\u0578\u0582\u0569\u0575\u0578\u0582\u0576 \u057d\u056f\u057d\u0565\u056c"
            },
            "ka": {
                "Bad": "\u10ea\u10e3\u10d3\u10d8",
                "Call": "\u10d6\u10d0\u10e0\u10d8",
                "Good": "\u10d9\u10d0\u10e0\u10d2\u10d8",
                "Name": "\u10e1\u10d0\u10ee\u10d4\u10da\u10d8",
                "Send": "\u10d2\u10d0\u10d2\u10d6\u10d0\u10d5\u10dc\u10d0",
                "Agree": "\u10d7\u10d0\u10dc\u10ee\u10db\u10dd\u10d1\u10d0",
                "Allow": "\u10dc\u10d4\u10d1\u10d0\u10e0\u10d7\u10d5\u10d0",
                "Loved": "\u10eb\u10d0\u10da\u10d8\u10d0\u10dc \u10d9\u10d0\u10e0\u10d2\u10d8",
                "Cancel": "\u10d2\u10d0\u10e3\u10e5\u10db\u10d4\u10d1\u10d0",
                "E-mail": "\u10d4\u10da-\u10e4\u10dd\u10e1\u10e2\u10d0",
                "Friday": "\u10de\u10d0\u10e0\u10d0\u10e1\u10d9\u10d4\u10d5\u10d8",
                "Mobile": "\u10db\u10dd\u10d1\u10d8\u10da\u10e3\u10e0\u10d8",
                "Monday": "\u10dd\u10e0\u10e8\u10d0\u10d1\u10d0\u10d7\u10d8",
                "Sunday": "\u10d9\u10d5\u10d8\u10e0\u10d0",
                "Average": "\u10e1\u10d0\u10e8\u10e3\u10d0\u10da\u10dd",
                "Message": "\u10e8\u10d4\u10e2\u10e7\u10dd\u10d1\u10d8\u10dc\u10d4\u10d1\u10d0",
                "Tuesday": "\u10e1\u10d0\u10db\u10e8\u10d0\u10d1\u10d0\u10d7\u10d8",
                "Operator": "\u10dd\u10de\u10d4\u10e0\u10d0\u10e2\u10dd\u10e0\u10d8",
                "Saturday": "\u10e8\u10d0\u10d1\u10d0\u10d7\u10d8",
                "Thursday": "\u10ee\u10e3\u10d7\u10e8\u10d0\u10d1\u10d0\u10d7\u10d8",
                "Very Bad": "\u10eb\u10d0\u10da\u10d8\u10d0\u10dc \u10ea\u10e3\u10d3\u10d8",
                "Free Call": "\u10e3\u10e4\u10d0\u10e1\u10dd \u10d6\u10d0\u10e0\u10d8",
                "Mute chat": "\u10ee\u10db\u10d8\u10e1 \u10e9\u10d0\u10e0\u10d7\u10d5\u10d0",
                "Wednesday": "\u10dd\u10d7\u10ee\u10e8\u10d0\u10d1\u10d0\u10d7\u10d8",
                "Start Chat": "\u10e1\u10d0\u10e3\u10d1\u10e0\u10d8\u10e1 \u10d3\u10d0\u10ec\u10e7\u10d4\u10d1\u10d0",
                "Choose Date": "\u10d0\u10d8\u10e0\u10e9\u10d8\u10d4\u10d7 \u10d7\u10d0\u10e0\u10d8\u10e6\u10d8",
                "Choose Time": "\u10d0\u10d8\u10e0\u10e9\u10d8\u10d4\u10d7 \u10d3\u10e0\u10dd",
                "Online Chat": "\u10e9\u10d0\u10e2\u10d8",
                "Unmute chat": "\u10ee\u10db\u10d8\u10e1 \u10d2\u10d0\u10db\u10dd\u10e0\u10d7\u10d5\u10d0",
                "Working Hours": "\u10e1\u10d0\u10db\u10e3\u10e8\u10d0\u10dd \u10e1\u10d0\u10d0\u10d7\u10d4\u10d1\u10d8",
                "Connecting ...": "\u10db\u10d8\u10db\u10d3\u10d8\u10dc\u10d0\u10e0\u10d4\u10dd\u10d1\u10e1 \u10d3\u10d0\u10d9\u10d0\u10d5\u10e8\u10d8\u10e0\u10d4\u10d1\u10d0 \u2026",
                "Add Your Number": "\u10e8\u10d4\u10d8\u10e7\u10d5\u10d0\u10dc\u10d4\u10d7 \u10d7\u10e5\u10d5\u10d4\u10dc\u10d8 \u10dc\u10dd\u10db\u10d4\u10e0\u10d8",
                "Thanks for rate": "\u10db\u10d0\u10d3\u10da\u10dd\u10d1\u10d0 \u10e8\u10d4\u10e4\u10d0\u10e1\u10d4\u10d1\u10d8\u10e1\u10d7\u10d5\u10d8\u10e1",
                "Allow Microphone": "\u10db\u10d8\u10d9\u10e0\u10dd\u10e4\u10dd\u10dc\u10d8\u10e1 \u10d2\u10d0\u10d0\u10e5\u10e2\u10d8\u10e3\u10e0\u10d4\u10d1\u10d0",
                "Request callback": "\u10d2\u10d0\u10d3\u10db\u10dd\u10e0\u10d4\u10d9\u10d5\u10d8\u10e1 \u10db\u10dd\u10d7\u10ee\u10dd\u10d5\u10dc\u10d0",
                "Back To Home Page": "\u10db\u10d7\u10d0\u10d5\u10d0\u10e0 \u10d2\u10d5\u10d4\u10e0\u10d3\u10d6\u10d4 \u10d3\u10d0\u10d1\u10e0\u10e3\u10dc\u10d4\u10d1\u10d0",
                "Choose department": "\u10d0\u10d8\u10e0\u10e9\u10d8\u10d4\u10d7 \u10d2\u10d0\u10dc\u10e7\u10dd\u10e4\u10d8\u10da\u10d4\u10d1\u10d0",
                "Type a message...": "\u10e8\u10d4\u10d8\u10e7\u10d5\u10d0\u10dc\u10d4\u10d7 \u10e8\u10d4\u10e2\u10e7\u10dd\u10d1\u10d8\u10dc\u10d4\u10d1\u10d0\u2026",
                "Co-browsing request": "\u10d7\u10d0\u10dc\u10d0-\u10d1\u10e0\u10d0\u10e3\u10d6\u10d8\u10dc\u10d2\u10d8\u10e1 \u10db\u10dd\u10d7\u10ee\u10dd\u10d5\u10dc\u10d0",
                "Now, we are offline": "\u10d0\u10db\u10df\u10d0\u10db\u10d0\u10d3 \u10e9\u10d5\u10d4\u10dc \u10d0\u10e0 \u10d5\u10d0\u10e0\u10d7 \u10ee\u10d0\u10d6\u10d6\u10d4",
                "Review conversation": "\u10e1\u10d0\u10e3\u10d1\u10e0\u10d8\u10e1 \u10d2\u10d0\u10d3\u10d0\u10ee\u10d4\u10d3\u10d5\u10d0",
                "You are in queue...": "\u10d7\u10e5\u10d5\u10d4\u10dc \u10ee\u10d0\u10e0\u10d7 \u10e0\u10d8\u10d2\u10e8\u10d8...",
                "Conversation is empty": "\u10e9\u10d0\u10e2\u10d8\u10e1 \u10e4\u10d0\u10dc\u10ef\u10d0\u10e0\u10d0 \u10ea\u10d0\u10e0\u10d8\u10d4\u10da\u10d8\u10d0",
                "One more step to call": "\u10d9\u10d8\u10d3\u10d4\u10d5 \u10d4\u10e0\u10d7\u10d8 \u10e5\u10db\u10d4\u10d3\u10d4\u10d1\u10d0 \u10d6\u10d0\u10e0\u10d8\u10e1 \u10d2\u10d0\u10dc\u10e1\u10d0\u10ee\u10dd\u10e0\u10ea\u10d8\u10d4\u10da\u10d4\u10d1\u10da\u10d0\u10d3",
                "One more step to chat": "\u10d9\u10d8\u10d3\u10d4\u10d5 \u10d4\u10e0\u10d7\u10d8 \u10e5\u10db\u10d4\u10d3\u10d4\u10d1\u10d0 \u10e9\u10d0\u10e2\u10d8\u10e1 \u10d3\u10d0\u10e1\u10d0\u10ec\u10e7\u10d4\u10d1\u10d0\u10d3",
                "Your message here ...": "\u10d3\u10d0\u10e2\u10dd\u10d5\u10d4\u10d7 \u10d0\u10e5 \u10d7\u10e5\u10d5\u10d4\u10dc\u10d8 \u10e8\u10d4\u10e2\u10e7\u10dd\u10d1\u10d8\u10dc\u10d4\u10d1\u10d0...",
                "How we are Helpfully ?": "\u10d2\u10d7\u10ee\u10dd\u10d5\u10d7 \u10e8\u10d4\u10d0\u10e4\u10d0\u10e1\u10dd\u10d7 \u10d2\u10d0\u10ec\u10d4\u10e3\u10da\u10d8 \u10db\u10dd\u10db\u10e1\u10d0\u10ee\u10e3\u10e0\u10d4\u10d1\u10d0",
                "Queue position: :position": "\u10d7\u10e5\u10d5\u10d4\u10dc \u10ee\u10d0\u10e0\u10d7 \u10e0\u10d8\u10d2\u10e8\u10d8 N :position",
                "Hi, we are here to help you": "\u10d2\u10d0\u10db\u10d0\u10e0\u10ef\u10dd\u10d1\u10d0",
                "Conversation will appear here": "\u10e1\u10d0\u10e3\u10d1\u10d0\u10e0\u10d8 \u10d2\u10d0\u10db\u10dd\u10e9\u10dc\u10d3\u10d4\u10d1\u10d0 \u10d0\u10e5",
                "Your conversation has ended !": "\u10d7\u10e5\u10d5\u10d4\u10dc\u10d8 \u10e1\u10d0\u10e3\u10d1\u10d0\u10e0\u10d8 \u10d3\u10d0\u10e1\u10e0\u10e3\u10da\u10d3\u10d0!",
                "please allow it to make a call.": "\u10d2\u10d7\u10ee\u10dd\u10d5\u10d7 \u10d2\u10d0\u10d0\u10e5\u10e2\u10d8\u10e3\u10e0\u10dd\u10d7 \u10dc\u10d4\u10d1\u10d0\u10e0\u10d7\u10d5\u10d0 \u10d6\u10d0\u10e0\u10d8\u10e1 \u10d2\u10d0\u10dc\u10e1\u10d0\u10ee\u10dd\u10e0\u10ea\u10d8\u10d4\u10da\u10d4\u10d1\u10da\u10d0\u10d3",
                "Your conversation has finished !": "\u10d7\u10e5\u10d5\u10d4\u10dc\u10d8 \u10e1\u10d0\u10e3\u10d1\u10d0\u10e0\u10d8 \u10d3\u10d0\u10e1\u10e0\u10e3\u10da\u10d3\u10d0!",
                "Chosen time should be in the future": "\u10d3\u10e0\u10dd \u10e3\u10dc\u10d3\u10d0 \u10d8\u10e7\u10dd\u10e1 \u10db\u10d8\u10d7\u10d8\u10d7\u10d4\u10d1\u10e3\u10da\u10d8 \u10db\u10dd\u10db\u10d0\u10d5\u10d0\u10da \u10d3\u10e6\u10d4\u10d4\u10d1\u10d6\u10d4.",
                "Maximum amount of callbacks reached": "\u10d2\u10d0\u10d3\u10db\u10dd\u10e0\u10d4\u10d9\u10d5\u10d8\u10e1 \u10da\u10d8\u10db\u10d8\u10e2\u10d8 \u10d0\u10db\u10dd\u10ec\u10e3\u10e0\u10e3\u10da\u10d8\u10d0",
                "The comment must be at least 3 characters.": "\u10d9\u10dd\u10db\u10d4\u10dc\u10e2\u10d0\u10e0\u10d8 \u10e3\u10dc\u10d3\u10d0 \u10db\u10dd\u10d8\u10ea\u10d0\u10d5\u10d3\u10d4\u10e1 \u10db\u10d8\u10dc\u10d8\u10db\u10e3\u10db 3 \u10e1\u10d8\u10db\u10d1\u10dd\u10da\u10dd\u10e1",
                "You have not allowed microphone permission": "\u10d7\u10e5\u10d5\u10d4\u10dc \u10d0\u10e0 \u10d2\u10d0\u10e5\u10d5\u10d7 \u10e9\u10d0\u10e0\u10d7\u10e3\u10da\u10d8 \u10db\u10d8\u10d9\u10e0\u10dd\u10e4\u10dd\u10dc\u10d6\u10d4 \u10ec\u10d5\u10d3\u10dd\u10db\u10d8\u10e1 \u10dc\u10d4\u10d1\u10d0\u10e0\u10d7\u10d5\u10d0",
                "The Phone Number must be at least 4 characters.": "\u10e2\u10d4\u10da\u10d4\u10e4\u10dd\u10dc\u10d8\u10e1 \u10dc\u10dd\u10db\u10d4\u10e0\u10d8 \u10e3\u10dc\u10d3\u10d0 \u10db\u10dd\u10d8\u10ea\u10d0\u10d5\u10d3\u10d4\u10e1 \u10db\u10d8\u10dc\u10d8\u10db\u10e3\u10db 4 \u10ea\u10d8\u10e4\u10e0\u10e1.",
                "Choose how you are comfortable with to contact us": "\u10d2\u10d0\u10e5\u10d5\u10d7 \u10d9\u10d8\u10d7\u10ee\u10d5\u10d4\u10d1\u10d8? \u10db\u10dd\u10d2\u10d5\u10ec\u10d4\u10e0\u10d4\u10d7 \u10d0\u10dc \u10d3\u10d0\u10d2\u10d5\u10d8\u10e0\u10d4\u10d9\u10d4\u10d71.",
                "Your request about callback have successfully sent": "\u10d7\u10e5\u10d5\u10d4\u10dc\u10d8 \u10d2\u10d0\u10d3\u10db\u10dd\u10e0\u10d4\u10d9\u10d5\u10d8\u10e1 \u10db\u10dd\u10d7\u10ee\u10dd\u10d5\u10dc\u10d0 \u10d3\u10d0\u10e4\u10d8\u10e5\u10e1\u10d8\u10e0\u10d4\u10d1\u10e3\u10da\u10d8\u10d0",
                "To start secure co-browsing session please click allow": "\u10e3\u10e1\u10d0\u10e4\u10e0\u10d7\u10ee\u10dd \u10d7\u10d0\u10dc\u10d0-\u10d1\u10e0\u10d0\u10e3\u10d6\u10d8\u10dc\u10d2\u10d8\u10e1 \u10e1\u10d4\u10e1\u10d8\u10d8\u10e1 \u10d3\u10d0\u10e1\u10d0\u10ec\u10e7\u10d4\u10d1\u10d0\u10d3 \u10d2\u10d7\u10ee\u10dd\u10d5\u10d7 \u10d3\u10d0\u10d0\u10ed\u10d8\u10e0\u10dd\u10d7 \u10e6\u10d8\u10da\u10d0\u10d9\u10e1 \u10dc\u10d4\u10d1\u10d0\u10e0\u10d7\u10d5\u10d0",
                "Choose your preferred time and date for us to contact you": "\u10d0\u10d8\u10e0\u10e9\u10d8\u10d4\u10d7 \u10e1\u10d0\u10e1\u10e3\u10e0\u10d5\u10d4\u10da\u10d8 \u10d3\u10e0\u10dd \u10d3\u10d0\u10e1\u10d0\u10d9\u10d0\u10d5\u10e8\u10d8\u10e0\u10d4\u10d1\u10da\u10d0\u10d3",
                "Fill the information to Start a conversation The team replies in a few hours": "\u10e9\u10d0\u10e2\u10d8\u10e1 \u10d3\u10d0\u10e1\u10d0\u10ec\u10e7\u10d4\u10d1\u10d0\u10d3 \u10e8\u10d4\u10d0\u10d5\u10e1\u10d4\u10d7 \u10d5\u10d4\u10da\u10d4\u10d1\u10d8, \u10e9\u10d5\u10d4\u10dc\u10d8 \u10e1\u10de\u10d4\u10ea\u10d8\u10d0\u10da\u10d8\u10e1\u10e2\u10d8 \u10db\u10d0\u10da\u10d4 \u10d2\u10d8\u10de\u10d0\u10e1\u10e3\u10ee\u10d4\u10d1\u10d7.",
                "The chat will be closed and the current conversation won't be accessible. You will be able to start a new conversation.": "\u10e9\u10d0\u10e2\u10d8\u10e1 \u10e4\u10d0\u10dc\u10ef\u10d0\u10e0\u10d0 \u10d3\u10d0\u10d8\u10ee\u10e3\u10e0\u10d4\u10d1\u10d0 \u10d3\u10d0 \u10db\u10d8\u10db\u10d3\u10d8\u10dc\u10d0\u10e0\u10d4 \u10e1\u10d0\u10e3\u10d1\u10d0\u10e0\u10d8 \u10d0\u10e6\u10d0\u10e0 \u10d8\u10e5\u10dc\u10d4\u10d1\u10d0 \u10ee\u10d4\u10da\u10db\u10d8\u10e1\u10d0\u10ec\u10d5\u10d3\u10dd\u10db\u10d8. \u10d7\u10e5\u10d5\u10d4\u10dc \u10e8\u10d4\u10e1\u10eb\u10da\u10d4\u10d1\u10d7 \u10d0\u10ee\u10d0\u10da\u10d8 \u10e1\u10d0\u10e3\u10d1\u10e0\u10d8\u10e1 \u10d3\u10d0\u10ec\u10e7\u10d4\u10d1\u10d0\u10e1."
            },
            "pl": {
                "Bad": "Z\u0142y",
                "Call": "Po\u0142\u0105czenie",
                "Good": "Dobrze",
                "Name": "Nazwa",
                "Send": "Wy\u015blij",
                "Agree": "Zgoda",
                "Allow": "Pozwalam",
                "Loved": "Super",
                "Cancel": "Rezygnacja",
                "E-mail": "E-mail",
                "Friday": "Pi\u0105tek",
                "Mobile": "Urzadzenie mobilne",
                "Monday": "Poniedzia\u0142ek",
                "Sunday": "Niedziela",
                "Average": "\u015aredni",
                "Message": "Wiadomo\u015b\u0107",
                "Tuesday": "Wtorek",
                "Operator": "Operator",
                "Saturday": "Saturday",
                "Thursday": "Czwartek",
                "Very Bad": "Bardzo \u017ale",
                "Free Call": "Darmowe po\u0142\u0105czenie",
                "Mute chat": "Wycisz czat",
                "Wednesday": "\u015aroda",
                "Start Chat": "Rozpocznij czat",
                "Choose Date": "Wybierz dat\u0119",
                "Choose Time": "Wybierz Czas",
                "Online Chat": "Czat online",
                "Unmute chat": "Wy\u0142\u0105cz wyciszenie czatu",
                "Working Hours": "Godziny pracy",
                "Connecting ...": "\u0141\u0105czenie...",
                "Add Your Number": "Dodaj sw\u00f3j numer",
                "Thanks for rate": "Dzi\u0119kuj\u0119 za opini\u0119",
                "Allow Microphone": "Zezw\u00f3l na mikrofon",
                "Request callback": "Popro\u015b o oddzwonienie",
                "Back To Home Page": "Powr\u00f3t na stron\u0119 g\u0142\u00f3wn\u0105",
                "Choose department": "Wybierz dzia\u0142",
                "Type a message...": "Wpisz wiadomo\u015b\u0107...",
                "Co-browsing request": "\u017b\u0105danie wsp\u00f3lnego przegl\u0105dania",
                "Now, we are offline": "Teraz jeste\u015bmy offline",
                "Review conversation": "Przejrzyj rozmow\u0119",
                "You are in queue...": "Jeste\u015b w kolejce...",
                "Conversation is empty": "Rozmowa jest pusta",
                "One more step to call": "Jeszcze jeden krok, by zadzwoni\u0107",
                "One more step to chat": "Jeszcze jeden krok do rozmowy",
                "Your message here ...": "Twoja wiadomo\u015b\u0107 tutaj ...",
                "How we are Helpfully ?": "Jak oceniasz otrzymane wsparcie?",
                "Queue position: :position": "Pozycja w kolejce: :pozycja",
                "Hi, we are here to help you": "Cze\u015b\u0107, jeste\u015bmy tutaj, aby Ci pom\u00f3c",
                "Conversation will appear here": "Tutaj pojawi si\u0119 rozmowa",
                "Your conversation has ended !": "Twoja rozmowa zosta\u0142a zako\u0144czona!",
                "please allow it to make a call.": "prosz\u0119 pozwoli\u0107 mu wykona\u0107 po\u0142\u0105czenie",
                "Your conversation has finished !": "Twoja rozmowa zosta\u0142a zako\u0144czona!",
                "Chosen time should be in the future": "Wybrany czas powinien by\u0107 w przysz\u0142o\u015bci",
                "Maximum amount of callbacks reached": "Osi\u0105gni\u0119to maksymaln\u0105 liczb\u0119 wywo\u0142a\u0144 zwrotnych",
                "The comment must be at least 3 characters.": "Komentarz musi mie\u0107 co najmniej 3 znaki.",
                "You have not allowed microphone permission": "Nie zezwoli\u0142e\u015b na dost\u0119p do mikrofonu",
                "The Phone Number must be at least 4 characters.": "Numer telefonu musi mie\u0107 co najmniej 4 znaki.",
                "Choose how you are comfortable with to contact us": "Wybierz spos\u00f3b, w jaki chcesz si\u0119 z nami skontaktowa\u0107",
                "Your request about callback have successfully sent": "Twoja pro\u015bba o oddzwonienie zosta\u0142a przes\u0142ana",
                "To start secure co-browsing session please click allow": "Aby rozpocz\u0105\u0107 bezpieczn\u0105 sesj\u0119 wsp\u00f3lnego przegl\u0105dania, kliknij przycisk Zezw\u00f3l",
                "Choose your preferred time and date for us to contact you": "Wybierz preferowan\u0105 godzin\u0119 i dat\u0119, aby\u015bmy oddzwonili",
                "Fill the information to Start a conversation The team replies in a few hours": "Uzupe\u0142nij informacje. Wkr\u00f3tce skontaktujemy si\u0119 z zespo\u0142em",
                "The chat will be closed and the current conversation won't be accessible. You will be able to start a new conversation.": "Czat zostanie zamkni\u0119ty, a bie\u017c\u0105ca rozmowa nie b\u0119dzie dost\u0119pna. B\u0119dziesz m\u00f3g\u0142 rozpocz\u0105\u0107 now\u0105 rozmow\u0119."
            },
            "ru": {
                "Bad": "\u041f\u043b\u043e\u0445\u043e",
                "Call": "\u0417\u0432\u043e\u043d\u043e\u043a",
                "Good": "\u0425\u043e\u0440\u043e\u0448\u043e",
                "Name": "\u0418\u043c\u044f",
                "Send": "\u041e\u0442\u043f\u0440\u0430\u0432\u0438\u0442\u044c",
                "Agree": "\u0421\u043e\u0433\u043b\u0430\u0441\u0438\u0435",
                "Allow": "\u0420\u0430\u0437\u0440\u0435\u0448\u0438\u0442\u044c",
                "Loved": "\u041e\u0447\u0435\u043d\u044c \u0445\u043e\u0440\u043e\u0448\u043e",
                "Cancel": "\u041e\u0442\u043c\u0435\u043d\u0430",
                "E-mail": "\u042d\u043b\u0435\u043a\u0442\u0440\u043e\u043d\u043d\u0430\u044f \u043f\u043e\u0447\u0442\u0430",
                "Friday": "\u041f\u044f\u0442\u043d\u0438\u0446\u0430",
                "Mobile": "\u041c\u043e\u0431\u0438\u043b\u044c\u043d\u044b\u0439",
                "Monday": "\u041f\u043e\u043d\u0435\u0434\u0435\u043b\u044c\u043d\u0438\u043a",
                "Sunday": "\u0412\u043e\u0441\u043a\u0440\u0435\u0441\u0435\u043d\u044c\u0435",
                "Average": "\u0421\u0440\u0435\u0434\u043d\u0435",
                "Message": "\u0421\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435",
                "Tuesday": "\u0412\u0442\u043e\u0440\u043d\u0438\u043a",
                "Operator": "\u041e\u043f\u0435\u0440\u0430\u0442\u043e\u0440",
                "Saturday": "\u0421\u0443\u0431\u0431\u043e\u0442\u0430",
                "Thursday": "\u0427\u0435\u0442\u0432\u0435\u0440\u0433",
                "Very Bad": "\u041e\u0447\u0435\u043d\u044c \u043f\u043b\u043e\u0445\u043e",
                "Free Call": "\u0411\u0435\u0441\u043f\u043b\u0430\u0442\u043d\u044b\u0439 \u0437\u0432\u043e\u043d\u043e\u043a",
                "Mute chat": "\u041e\u0442\u043a\u043b\u044e\u0447\u0438\u0442\u044c \u0437\u0432\u0443\u043a",
                "Wednesday": "\u0421\u0440\u0435\u0434\u0430",
                "Start Chat": "\u041d\u0430\u0447\u0430\u0442\u044c \u0440\u0430\u0437\u0433\u043e\u0432\u043e\u0440",
                "Choose Date": "\u0412\u044b\u0431\u0435\u0440\u0438\u0442\u0435 \u0434\u0430\u0442\u0443",
                "Choose Time": "\u0412\u044b\u0431\u0435\u0440\u0438\u0442\u0435 \u0432\u0440\u0435\u043c\u044f",
                "Online Chat": "\u0427\u0430\u0442",
                "Unmute chat": "\u0412\u043a\u043b\u044e\u0447\u0438\u0442\u044c \u0437\u0432\u0443\u043a",
                "Working Hours": "\u0420\u0430\u0431\u043e\u0447\u0435\u0435 \u0432\u0440\u0435\u043c\u044f",
                "Connecting ...": "\u0418\u0434\u0435\u0442 \u043f\u043e\u0434\u043a\u043b\u044e\u0447\u0435\u043d\u0438\u0435...",
                "Add Your Number": "\u0412\u0432\u0435\u0434\u0438\u0442\u0435 \u0441\u0432\u043e\u0439 \u043d\u043e\u043c\u0435\u0440",
                "Thanks for rate": "\u0421\u043f\u0430\u0441\u0438\u0431\u043e \u0437\u0430 \u043e\u0446\u0435\u043d\u043a\u0443",
                "Allow Microphone": "\u0412\u043a\u043b\u044e\u0447\u0438\u0442\u044c \u043c\u0438\u043a\u0440\u043e\u0444\u043e\u043d",
                "Request callback": "\u0417\u0430\u043f\u0440\u043e\u0441 \u043e\u0431\u0440\u0430\u0442\u043d\u043e\u0433\u043e \u0437\u0432\u043e\u043d\u043a\u0430.",
                "Back To Home Page": "\u0412\u0435\u0440\u043d\u0443\u0442\u044c\u0441\u044f \u043d\u0430 \u0433\u043b\u0430\u0432\u043d\u0443\u044e \u0441\u0442\u0440\u0430\u043d\u0438\u0446\u0443",
                "Choose department": "\u0412\u044b\u0431\u0435\u0440\u0438\u0442\u0435 \u0440\u0430\u0437\u0434\u0435\u043b",
                "Type a message...": "\u0412\u0432\u0435\u0434\u0438\u0442\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435\u2026",
                "Co-browsing request": "\u0417\u0430\u043f\u0440\u043e\u0441 \u043d\u0430 \u0441\u043e\u0432\u043c\u0435\u0441\u0442\u043d\u044b\u0439 \u043f\u0440\u043e\u0441\u043c\u043e\u0442\u0440",
                "Now, we are offline": "\u0412 \u043d\u0430\u0441\u0442\u043e\u044f\u0449\u0435\u0435 \u0432\u0440\u0435\u043c\u044f \u043c\u044b \u043d\u0435 \u0432 \u0441\u0435\u0442\u0438",
                "Review conversation": "\u041e\u0431\u0437\u043e\u0440 \u0440\u0430\u0437\u0433\u043e\u0432\u043e\u0440\u0430",
                "You are in queue...": "\u0412\u044b \u0432 \u043e\u0447\u0435\u0440\u0435\u0434\u0438\u2026",
                "Conversation is empty": "\u041e\u043a\u043d\u043e \u0447\u0430\u0442\u0430 \u043f\u0443\u0441\u0442\u043e",
                "One more step to call": "\u0415\u0449\u0435 \u043e\u0434\u043d\u043e \u0434\u0435\u0439\u0441\u0442\u0432\u0438\u0435 \u0434\u043b\u044f \u0441\u043e\u0432\u0435\u0440\u0448\u0435\u043d\u0438\u044f \u0437\u0432\u043e\u043d\u043a\u0430",
                "One more step to chat": "\u0415\u0449\u0435 \u043e\u0434\u043d\u043e \u0434\u0435\u0439\u0441\u0442\u0432\u0438\u0435, \u0447\u0442\u043e\u0431\u044b \u043d\u0430\u0447\u0430\u0442\u044c \u0447\u0430\u0442",
                "Your message here ...": "\u041e\u0441\u0442\u0430\u0432\u044c\u0442\u0435 \u0441\u0432\u043e\u0435 \u0441\u043e\u043e\u0431\u0449\u0435\u043d\u0438\u0435 \u0437\u0434\u0435\u0441\u044c\u2026",
                "How we are Helpfully ?": "\u041f\u043e\u0436\u0430\u043b\u0443\u0439\u0441\u0442\u0430, \u043e\u0446\u0435\u043d\u0438\u0442\u0435 \u043f\u0440\u0435\u0434\u043e\u0441\u0442\u0430\u0432\u043b\u0435\u043d\u043d\u0443\u044e \u0443\u0441\u043b\u0443\u0433\u0443",
                "Queue position: :position": "\u0412\u044b \u0432 \u043e\u0447\u0435\u0440\u0435\u0434\u0438 N :position",
                "Hi, we are here to help you": "",
                "Conversation will appear here": "\u0417\u0434\u0435\u0441\u044c \u043f\u043e\u044f\u0432\u0438\u0442\u0441\u044f \u0440\u0430\u0437\u0433\u043e\u0432\u043e\u0440",
                "Your conversation has ended !": "\u0412\u0430\u0448 \u0440\u0430\u0437\u0433\u043e\u0432\u043e\u0440 \u043e\u043a\u043e\u043d\u0447\u0435\u043d!",
                "please allow it to make a call.": "\u041f\u043e\u0436\u0430\u043b\u0443\u0439\u0441\u0442\u0430, \u0430\u043a\u0442\u0438\u0432\u0438\u0440\u0443\u0439\u0442\u0435 \u0440\u0430\u0437\u0440\u0435\u0448\u0435\u043d\u0438\u0435 \u043d\u0430 \u0437\u0432\u043e\u043d\u043e\u043a",
                "Your conversation has finished !": "\u0412\u0430\u0448 \u0440\u0430\u0437\u0433\u043e\u0432\u043e\u0440 \u043e\u043a\u043e\u043d\u0447\u0435\u043d!",
                "Chosen time should be in the future": "\u041d\u0435\u043e\u0431\u0445\u043e\u0434\u0438\u043c\u043e \u0443\u043a\u0430\u0437\u0430\u0442\u044c \u0432\u0440\u0435\u043c\u044f \u043d\u0430 \u0431\u043b\u0438\u0436\u0430\u0439\u0448\u0438\u0435 \u0434\u043d\u0438",
                "Maximum amount of callbacks reached": "\u041b\u0438\u043c\u0438\u0442 \u0437\u0432\u043e\u043d\u043a\u043e\u0432 \u0438\u0441\u0447\u0435\u0440\u043f\u0430\u043d",
                "The comment must be at least 3 characters.": "\u041a\u043e\u043c\u043c\u0435\u043d\u0442\u0430\u0440\u0438\u0439 \u0434\u043e\u043b\u0436\u0435\u043d \u0441\u043e\u0434\u0435\u0440\u0436\u0430\u0442\u044c \u043d\u0435 \u043c\u0435\u043d\u0435\u0435 3 \u0441\u0438\u043c\u0432\u043e\u043b\u043e\u0432",
                "You have not allowed microphone permission": "\u0423 \u0412\u0430\u0441 \u043d\u0435\u0442 \u0440\u0430\u0437\u0440\u0435\u0448\u0435\u043d\u0438\u044f \u043d\u0430 \u0434\u043e\u0441\u0442\u0443\u043f \u043a \u043c\u0438\u043a\u0440\u043e\u0444\u043e\u043d\u0443",
                "The Phone Number must be at least 4 characters.": "\u041d\u043e\u043c\u0435\u0440 \u0442\u0435\u043b\u0435\u0444\u043e\u043d\u0430 \u0434\u043e\u043b\u0436\u0435\u043d \u0441\u043e\u0434\u0435\u0440\u0436\u0430\u0442\u044c \u043d\u0435 \u043c\u0435\u043d\u0435\u0435 4 \u0446\u0438\u0444\u0440",
                "Choose how you are comfortable with to contact us": "\u0423 \u0432\u0430\u0441 \u0432\u043e\u043f\u0440\u043e\u0441\u044b? \u041d\u0430\u043f\u0438\u0448\u0438\u0442\u0435 \u0438\u043b\u0438 \u043f\u043e\u0437\u0432\u043e\u043d\u0438\u0442\u0435 \u043d\u0430\u043c.",
                "Your request about callback have successfully sent": "",
                "To start secure co-browsing session please click allow": "\u0427\u0442\u043e\u0431\u044b \u043d\u0430\u0447\u0430\u0442\u044c \u0431\u0435\u0437\u043e\u043f\u0430\u0441\u043d\u044b\u0439 \u0441\u0435\u0430\u043d\u0441 \u0441\u043e\u0432\u043c\u0435\u0441\u0442\u043d\u043e\u0433\u043e \u043f\u0440\u043e\u0441\u043c\u043e\u0442\u0440\u0430, \u043d\u0430\u0436\u043c\u0438\u0442\u0435 \u043a\u043d\u043e\u043f\u043a\u0443 \u00ab\u0420\u0430\u0437\u0440\u0435\u0448\u0438\u0442\u044c\u00bb",
                "Choose your preferred time and date for us to contact you": "\u0412\u044b\u0431\u0435\u0440\u0438\u0442\u0435 \u043f\u0440\u0435\u0434\u043f\u043e\u0447\u0442\u0438\u0442\u0435\u043b\u044c\u043d\u043e\u0435 \u0432\u0440\u0435\u043c\u044f \u0438 \u0434\u0430\u0442\u0443, \u0447\u0442\u043e\u0431\u044b \u0441\u0432\u044f\u0437\u0430\u0442\u044c\u0441\u044f \u0441 \u0412\u0430\u043c\u0438",
                "Fill the information to Start a conversation The team replies in a few hours": "\u0417\u0430\u043f\u043e\u043b\u043d\u0438\u0442\u0435 \u043f\u043e\u043b\u044f, \u0447\u0442\u043e\u0431\u044b \u043d\u0430\u0447\u0430\u0442\u044c \u0447\u0430\u0442, \u043d\u0430\u0448 \u0441\u043f\u0435\u0446\u0438\u0430\u043b\u0438\u0441\u0442 \u043e\u0442\u0432\u0435\u0442\u0438\u0442 \u0432\u0430\u043c \u0432 \u0431\u043b\u0438\u0436\u0430\u0439\u0448\u0435\u0435 \u0432\u0440\u0435\u043c\u044f.",
                "The chat will be closed and the current conversation won't be accessible. You will be able to start a new conversation.": "\u041e\u043a\u043d\u043e \u0447\u0430\u0442\u0430 \u0437\u0430\u043a\u0440\u043e\u0435\u0442\u0441\u044f, \u0438 \u0442\u0435\u043a\u0443\u0449\u0438\u0439 \u0440\u0430\u0437\u0433\u043e\u0432\u043e\u0440 \u0441\u0442\u0430\u043d\u0435\u0442 \u043d\u0435\u0434\u043e\u0441\u0442\u0443\u043f\u0435\u043d. \u0412\u044b \u0441\u043c\u043e\u0436\u0435\u0442\u0435 \u043d\u0430\u0447\u0430\u0442\u044c \u043d\u043e\u0432\u044b\u0439 \u0440\u0430\u0437\u0433\u043e\u0432\u043e\u0440."
            },
            "tr": {
                "Bad": "Bad",
                "Call": "Call",
                "Good": "Good",
                "Name": "Name",
                "Send": "Send",
                "Agree": "Agree",
                "Allow": "Allow",
                "Loved": "Loved",
                "Cancel": "Cancel",
                "E-mail": "E-mail",
                "Friday": "Friday",
                "Mobile": "Mobile",
                "Monday": "Monday",
                "Sunday": "Sunday",
                "Average": "Average",
                "Message": "Message",
                "Tuesday": "Tuesday",
                "Operator": "Operator",
                "Saturday": "Saturday",
                "Thursday": "Thursday",
                "Very Bad": "Very Bad",
                "Free Call": "Free Call",
                "Mute chat": "Mute chat",
                "Wednesday": "Wednesday",
                "Start Chat": "Start Chat",
                "Choose Date": "Choose Date",
                "Choose Time": "Choose Time",
                "Online Chat": "Online Chat",
                "Unmute chat": "Unmute chat",
                "Working Hours": "Working Hours",
                "Connecting ...": "Connecting ...",
                "Add Your Number": "Add your number",
                "Thanks for rate": "Thanks for your feedback",
                "Allow Microphone": "Allow microphone",
                "Request callback": "Request a callback",
                "Back To Home Page": "Back to home page",
                "Choose department": "Choose department",
                "Type a message...": "Type a message...",
                "Co-browsing request": "Co-browsing request",
                "Now, we are offline": "Now, we are offline",
                "Review conversation": "Review conversation",
                "You are in queue...": "You are in queue...",
                "Conversation is empty": "Conversation is empty",
                "One more step to call": "One more step to call",
                "One more step to chat": "One more step to chat",
                "Your message here ...": "Your message here ...",
                "How we are Helpfully ?": "How would you rate the support you received?",
                "Queue position: :position": "Queue position: :position",
                "Hi, we are here to help you": "Hi, we are here to help you",
                "Conversation will appear here": "Conversation will appear here",
                "Your conversation has ended !": "Your conversation has ended !",
                "please allow it to make a call.": "please allow it to make a call",
                "Your conversation has finished !": "Your conversation has finished !",
                "Chosen time should be in the future": "Chosen time should be in the future",
                "Maximum amount of callbacks reached": "Maximum amount of callbacks reached",
                "The comment must be at least 3 characters.": "The comment must be at least 3 characters.",
                "You have not allowed microphone permission": "You have not allowed microphone permission",
                "The Phone Number must be at least 4 characters.": "The phone number must be at least 4 characters.",
                "Choose how you are comfortable with to contact us": "Select how you would like to contact us",
                "Your request about callback have successfully sent": "Your request about callback has been submitted",
                "To start secure co-browsing session please click allow": "To start a secure co-browsing session please click allow",
                "Choose your preferred time and date for us to contact you": "Choose your preferred time and date for us to call you back",
                "Fill the information to Start a conversation The team replies in a few hours": "Fill in the information. The team will be in touch shortly",
                "The chat will be closed and the current conversation won't be accessible. You will be able to start a new conversation.": "The chat will be closed and the current conversation won't be accessible. You will be able to start a new conversation."
            }
        },
        "trigger": null
    }
}
2.  POST - https://api.livecaller.io/v1/widget/auth/login
in payload we sent device parameters: payload: 
{"device":{"ua":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/148.0.0.0 Safari/537.36","browser":{"name":"Chrome","version":"148.0.0.0"},"os":{"name":"macOS","version":"10.15.7","versionName":"Catalina"},"platform":{"type":"desktop","vendor":"Apple"},"engine":{"name":"Blink"}}}

in response we get:

{
    "data": {
        "uuid": "e319d1e7-e4df-4798-87bc-2a7453179284",
        "sip": {
            "transportOptions": {
                "wsServers": [
                    "wss:\/\/sip.livecaller.io:8089\/ws"
                ]
            },
            "uri": "sip:984770645797769217@sip.livecaller.io",
            "username": "984770645797769217",
            "realm": "sip.livecaller.io",
            "password": "dASlWcUxzri6zJrD",
            "register": true,
            "autostart": true,
            "autostop": true
        },
        "visitor": {
            "uuid": "73394e0c-f0ad-4151-bdaf-3d8392342cba",
            "name": "Visitor 6718481",
            "mobile": null,
            "email": null,
            "user_id": null,
            "settings": {
                "notification_sounds": true
            }
        }
    },
    "auth": {
        "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwczpcL1wvYXBpLmxpdmVjYWxsZXIuaW9cL3YxXC93aWRnZXRcL2F1dGhcL2xvZ2luIiwiaWF0IjoxNzgxMDg4NDE4LCJleHAiOjE3ODg4NjQ0MTgsIm5iZiI6MTc4MTA4ODQxOCwianRpIjoiRmRLUWlKdExsZXpWUExKMSIsInN1YiI6IjczMzk0ZTBjLWYwYWQtNDE1MS1iZGFmLTNkODM5MjM0MmNiYSIsInBydiI6IjA4NDUzNDRmNzYyZDhmZjU1MDQ0N2FjOGQ0NWRiODE2OTNmZmRjZWYiLCJzaWQiOiJlMzE5ZDFlNy1lNGRmLTQ3OTgtODdiYy0yYTc0NTMxNzkyODQifQ.PMk8avK843aI35mxvRMfZiL5ZgAs4C1X7hlz2z-o_tk",
        "token_type": "Bearer",
        "expires_in": 7776000
    }
}

3. if I refresh GET  https://api.livecaller.io/v1/widget/auth/visitor  this api request is sent skiping step #2 somhow in the code we know that visitor is authenticated and in reponse we have: 
{
    "data": {
        "uuid": "e319d1e7-e4df-4798-87bc-2a7453179284",
        "sip": {
            "transportOptions": {
                "wsServers": [
                    "wss:\/\/sip.livecaller.io:8089\/ws"
                ]
            },
            "uri": "sip:984770645797769217@sip.livecaller.io",
            "username": "984770645797769217",
            "realm": "sip.livecaller.io",
            "password": "dASlWcUxzri6zJrD",
            "register": true,
            "autostart": true,
            "autostop": true
        },
        "visitor": {
            "uuid": "73394e0c-f0ad-4151-bdaf-3d8392342cba",
            "name": "Visitor 6718481",
            "mobile": null,
            "email": null,
            "user_id": null,
            "settings": {
                "notification_sounds": true
            }
        }
    },
    "auth": {
        "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwczpcL1wvYXBpLmxpdmVjYWxsZXIuaW9cL3YxXC93aWRnZXRcL2F1dGhcL3Zpc2l0b3IiLCJpYXQiOjE3ODEwODg0MTgsImV4cCI6MTc4ODg2NDU4MCwibmJmIjoxNzgxMDg4NTgwLCJqdGkiOiJNQjhFWDduS0JLVUNkRlZ5Iiwic3ViIjoiNzMzOTRlMGMtZjBhZC00MTUxLWJkYWYtM2Q4MzkyMzQyY2JhIiwicHJ2IjoiMDg0NTM0NGY3NjJkOGZmNTUwNDQ3YWM4ZDQ1ZGI4MTY5M2ZmZGNlZiIsInNpZCI6ImUzMTlkMWU3LWU0ZGYtNDc5OC04N2JjLTJhNzQ1MzE3OTI4NCJ9.UlU7ax-pl8d9hZpHDp2LcMZD9Jo2hyzFVjOjeUep3aM",
        "token_type": "Bearer",
        "expires_in": 7776000
    }
}



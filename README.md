#!/usr/bin/env python3

import requests
import random
import json
import sys
from local import *

resp_js = None
is_private = True
total_uploads = 12

def proxy_session():
    session = requests.session()
    session.proxies = {
        'http':  'socks5://127.0.0.1:9050',
        'https': 'socks5://127.0.0.1:9050'
    }
    return session

def get_page(usrname):
    global resp_js
    session = requests.session()
    session.headers = {'User-Agent': random.choice(useragent)}
    resp_js = session.get('https://www.instagram.com/'+usrname+'/?__a=1').text
    return resp_js

def exinfo():
    def xprint(xdict, text):
        if xdict:
            print(f"{su} {re}most used {text} :")
            i = 0
            for key, val in xdict.items():
                if len(mail) == 1 and key in mail[0]:
                    continue
                print(f"  {gr}{key} : {wh}{val}")
                i += 1
                if i > 4:
                    break
            print()
    raw = find(resp_js)
    mail = raw.get('email', [])
    tags = sort_list(raw['tags'])
    ment = sort_list(raw['mention'])

    if mail:
        if len(mail) == 1:
            print(f"{su} {re}email found :\n{gr}  {mail[0]}")
        else:
            print(f"{su} {re}email found :")
            for x in mail:
                print(f"{gr}  {x}")
        print()

    xprint(tags, "tags")
    xprint(ment, "mentions")

def user_info(usrname):
    global resp_js, total_uploads, is_private
    resp_js = get_page(usrname)
    js = json.loads(resp_js)
    js = js['graphql']['user']

    is_private = js['is_private']

    if js['edge_owner_to_timeline_media']['count'] > 12:
        pass
    else:
        total_uploads = js['edge_owner_to_timeline_media']['count']

    usrinfo = {
        'username': js['username'],
        'user id': js['id'],
        'name': js['full_name'],
        'followers': js['edge_followed_by']['count'],
        'following': js['edge_follow']['count'],
        'posts img': js['edge_owner_to_timeline_media']['count'],
        'posts vid': js['edge_felix_video_timeline']['count'],
        'reels': js['highlight_reel_count'],
        'bio': js['biography'].replace('\n', ', '),
        'external url': js['external_url'],
        'private': js['is_private'],
        'verified': js['is_verified'],
        'profile img': urlshortner(js['profile_pic_url_hd']),
        'business account': js['is_business_account'],
        'joined recently': js['is_joined_recently'],
        'business category': js['business_category_name'],
        'category': js['category_enum'],
        'has guides': js['has_guides'],
    }

    banner()

    print(f"{su}{re} user info")
    for key, val in usrinfo.items():
        print(f"  {gr}{key} : {wh}{val}")

    print()

    exinfo()

def highlight_post_info(i):
    postinfo = {}
    total_child = 0
    child_img_list = []

    x = json.loads(resp_js)
    js = x['graphql']['user']['edge_owner_to_timeline_media']['edges'][i]['node']

    info = {
        'comments': js['edge_media_to_comment']['count'],
        'comment disable': js['comments_disabled'],
        'timestamp': js['taken_at_timestamp'],
        'likes': js['edge_liked_by']['count'],
        'location': js['location'],
    }

    try:
        info['caption'] = js['edge_media_to_caption']['edges'][0]['node']['text']
    except IndexError:
        pass

    if 'edge_sidecar_to_children' in js:
        total_child = len(js['edge_sidecar_to_children']['edges'])

        for child in range(total_child):
            js = x['graphql']['user']['edge_owner_to_timeline_media']['edges'][i]['node']['edge_sidecar_to_children']['edges'][child]['node']
            img_info = {
                'typename': js['__typename'],                 'id': js['id'],
                'shortcode': js['shortcode'],
                'dimensions': str(js['dimensions']['height'] + js['dimensions']['width']),
                'image url': js['display_url'],
                'fact check overall': js['fact_check_overall_rating'],
                'fact check': js['fact_check_information'],
                'gating info': js['gating_info'],
                'media overlay info': js['media_overlay_info'],
                'is_video': js['is_video'],
                'accessibility': js['accessibility_caption']
            }

            child_img_list.append(img_info)

        postinfo['imgs'] = child_img_list
        postinfo['info'] = info

    else:
        img_info = {
            'typename': js['__typename'],
            'id': js['id'],
            'shortcode': js['shortcode'],
            'dimensions': str(js['dimensions']['height'] + js['dimensions']['width']),
            'image url': js['display_url'],
            'fact check overall': js['fact_check_overall_rating'],
            'fact check': js['fact_check_information'],
            'gating info': js['gating_info'],
            'media overlay info': js['media_overlay_info'],
            'is_video': js['is_video'],
            'accessibility': js['accessibility_caption']
        }

        child_img_list.append(img_info)

        postinfo['imgs'] = child_img_list
        postinfo['info'] = info

    return postinfo

def post_info():

    if is_private:
        print(f"{fa} {gr}cannot use -p for private accounts !\n")
        sys.exit(1)

    posts = []

    for x in range(total_uploads):
        posts.append(highlight_post_info(x))

    for x in range(len(posts)):
        print(f"{su}{re} post {x} :")
        for key, val in posts[x].items():
            if key == 'imgs':
                postlen = len(val)
                print(f"{su}{re} contains {postlen} media")
                for y in range(postlen):
                    for xkey, xval in val[y].items():
                        print(f"  {gr}{xkey} : {wh}{xval}")
            if key == 'info':
                print(f"{su}{re} info :")
                for key, val in val.items():
                    print(f"  {gr}{key} : {wh}{val}")
                print("")

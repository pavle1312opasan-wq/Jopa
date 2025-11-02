import pygame, math, sys
from itertools import chain

pygame.init()
WIDTH, HEIGHT = 1000, 700
screen = pygame.display.set_mode((WIDTH, HEIGHT))
clock = pygame.time.Clock()
FPS = 60

# Walls: list of segments ((x1,y1),(x2,y2))
walls = [
    ((100,100),(400,120)),
    ((450,80),(450,400)),
    ((600,150),(900,120)),
    ((200,300),(350,550)),
    ((50,600),(300,500)),
    ((700,400),(950,650)),
    ((500,500),(700,520)),
    # rectangle room borders
    ((0,0),(WIDTH,0)),
    ((WIDTH,0),(WIDTH,HEIGHT)),
    ((WIDTH,HEIGHT),(0,HEIGHT)),
    ((0,HEIGHT),(0,0))
]

# Utility: intersection between ray (p + t*r, t>=0) and segment (q + u*s, u in [0,1])
def intersect_ray_segment(px, py, dx, dy, x1, y1, x2, y2):
    # Ray: P + t * D
    # Segment: A + u * (B - A)
    rx, ry = dx, dy
    sx, sy = x2 - x1, y2 - y1
    denom = rx * sy - ry * sx
    if abs(denom) < 1e-8:
        return None  # parallel
    t = ((x1 - px) * sy - (y1 - py) * sx) / denom
    u = ((x1 - px) * ry - (y1 - py) * rx) / denom
    if t >= 0 and 0 <= u <= 1:
        ix = px + t * rx
        iy = py + t * ry
        dist = math.hypot(ix - px, iy - py)
        return (ix, iy, dist, t)
    return None

def get_visibility_polygon(px, py, segments):
    # Collect unique segment endpoints
    points = set()
    for a,b in segments:
        points.add(a)
        points.add(b)
    pts = list(points)

    # For each point, cast rays at angle- eps, angle, +eps to avoid missing edges
    EPS = 1e-4
    ray_angles = []
    for (qx,qy) in pts:
        angle = math.atan2(qy - py, qx - px)
        ray_angles += [angle - EPS, angle, angle + EPS]

    # For each ray angle, find nearest intersection
    intersects = []
    for angle in ray_angles:
        dx = math.cos(angle)
        dy = math.sin(angle)
        nearest = None
        min_dist = float('inf')
        for seg in segments:
            res = intersect_ray_segment(px, py, dx, dy, seg[0][0], seg[0][1], seg[1][0], seg[1][1])
            if res:
                ix,iy,dist,t = res
                if dist < min_dist:
                    min_dist = dist
                    nearest = (ix,iy,angle,dist)
        if nearest:
            intersects.append(nearest)

    # Remove duplicates by rounding coords, then sort by angle
    unique = {}
    for ix,iy,angle,dist in intersects:
        key = (round(ix,4), round(iy,4))
        if key not in unique or unique[key][3] > dist:
            unique[key] = (ix,iy,angle,dist)
    verts = list(unique.values())
    verts.sort(key=lambda v: math.atan2(v[1]-py, v[0]-px))
    polygon = [(v[0], v[1]) for v in verts]
    return polygon

def draw():
    screen.fill((30,30,30))
    mx,my = pygame.mouse.get_pos()

    # draw walls
    for a,b in walls:
        pygame.draw.line(screen, (200,200,200), a, b, 2)

    # compute visibility polygon
    poly = get_visibility_polygon(mx, my, walls)

    # draw visible area
    if len(poly) >= 3:
        pygame.draw.polygon(screen, (255,230,100,120), poly)  # filled
        # outline
        pygame.draw.polygon(screen, (255,200,0), poly, 2)

    # draw player
    pygame.draw.circle(screen, (50,200,255), (mx,my), 6)
    pygame.draw.circle(screen, (50,200,255), (mx,my), 2)

    # debug: draw rays (optional, commented out)
    # for x,y in poly:
    #     pygame.draw.line(screen, (100,100,255), (mx,my), (x,y), 1)

    # HUD
    font = pygame.font.SysFont(None, 20)
    txt = font.render("Move mouse to see visibility polygon (educational demo)", True, (220,220,220))
    screen.blit(txt, (8,8))

running = True
while running:
    for ev in pygame.event.get():
        if ev.type == pygame.QUIT:
            running = False
        elif ev.type == pygame.KEYDOWN:
            if ev.key == pygame.K_ESCAPE:
                running = False

    draw()
    pygame.display.flip()
    clock.tick(FPS)

pygame.quit()
sys.exit()

